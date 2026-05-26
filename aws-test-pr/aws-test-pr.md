---
name: aws-test-pr
description: Test a cloud-api-adaptor GitHub PR against an existing EKS peer-pods cluster. Checks out the PR branch, detects what needs rebuilding (CAA image, podvm image, or both), builds and pushes artifacts, deploys to the running cluster, runs a verification pod, and supports selective cleanup.
argument-hint: "<PR_NUMBER> [--caa] [--podvm] [--rebuild-podvm] [--no-build] [--test-filter <regex>] [--cleanup]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - WebFetch
  - AskUserQuestion
---

<objective>
Test a cloud-api-adaptor GitHub PR against an existing EKS peer-pods cluster:

1. **Inspect** — fetch PR metadata and diff to determine what components changed
2. **Checkout** — fetch and checkout the PR branch in the local repo
3. **Build** — rebuild only what changed (CAA binary image, podvm image, or both)
4. **Deploy** — push new CAA image to ECR and patch the daemonset; create new AMI and update the peer-pods-cm configmap
5. **Test** — run a verification pod and confirm it creates an EC2 peer-pod VM
6. **Cleanup** — selectively revert: delete new AMI, restore original CAA image, remove test pods

`--rebuild-podvm` is an iteration flag for when you need to rebuild and re-register only the podvm AMI without touching the CAA binary or recreating the cluster. It deregisters the current PR AMI, rebuilds `system.raw` from the checked-out PR branch, registers a fresh AMI, updates `peer-pods-cm`, and does a clean CAA restart — then re-runs the smoke test.

AWS credentials assumed in environment. kubectl context assumed to be the EKS cluster from `/aws-eks-caa setup`.
</objective>

<repo_layout>
```
~/cloud-api-adaptor/
└── src/
    ├── cloud-api-adaptor/     ← CAA_ROOT — CAA binary, podvm-mkosi
    └── cloud-providers/       ← separate Go module, compiled into CAA image
```

Key artifact paths:
- CAA container image  → ECR: `<account>.dkr.ecr.<region>.amazonaws.com/cloud-api-adaptor`
- PodVM raw image      → `CAA_ROOT/podvm-mkosi/build/system.raw`
- PodVM AMI           → registered via `/aws-podvm-ami upload`
- Cluster config      → `peer-pods-cm` ConfigMap in `confidential-containers-system`
- CAA daemonset image → patched via `kubectl set image` or `helm upgrade`
</repo_layout>

<process>

## 0. Parse Arguments

Parse `$ARGUMENTS`:
- First positional arg: PR number (required)
- `--caa` — force rebuild of the CAA container image even if diff doesn't require it
- `--podvm` — force rebuild of the podvm image/AMI even if diff doesn't require it
- `--rebuild-podvm` — iteration flag: deregister the existing PR AMI, rebuild podvm from the checked-out branch, register a new AMI, update peer-pods-cm, clean-restart CAA, re-run the smoke test. Does NOT touch the CAA binary image. Requires a prior run to have set `/tmp/caa_pr_*` state.
- `--no-build` — skip all builds; just re-deploy and test with whatever images exist
- `--test-filter <regex>` — pass a Go test regex to `go test -run` for e2e tests instead of the default nginx smoke test
- `--cleanup` — after testing, run the cleanup subcommand automatically

---

## 1. Resolve environment

```bash
export AWS_PROFILE="${AWS_PROFILE:-osc}"
CAA_ROOT=~/cloud-api-adaptor/src/cloud-api-adaptor
REGION=$(cat /tmp/caa_eks_region 2>/dev/null || echo "us-east-2")
KUBECONFIG_FILE=$(cat /tmp/caa_eks_kubeconfig 2>/dev/null)
export KUBECONFIG="$KUBECONFIG_FILE"

ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
ECR_REPO="${ECR_REGISTRY}/cloud-api-adaptor"

[[ -f "$KUBECONFIG_FILE" ]] || { echo "ERROR: No kubeconfig found — run /aws-eks-caa setup first"; exit 1; }
kubectl get daemonset cloud-api-adaptor-daemonset -n confidential-containers-system &>/dev/null \
    || { echo "ERROR: CAA daemonset not found — run /aws-eks-caa setup first"; exit 1; }
```

Record the current (pre-PR) state so it can be restored during cleanup:
```bash
ORIGINAL_CAA_IMAGE=$(kubectl get daemonset cloud-api-adaptor-daemonset \
    -n confidential-containers-system \
    -o jsonpath='{.spec.template.spec.containers[0].image}')
ORIGINAL_AMI_ID=$(kubectl get configmap peer-pods-cm \
    -n confidential-containers-system \
    -o jsonpath='{.data.PODVM_AMI_ID}')

echo "Original CAA image: $ORIGINAL_CAA_IMAGE"
echo "Original AMI ID:    $ORIGINAL_AMI_ID"

echo "$ORIGINAL_CAA_IMAGE" > /tmp/caa_pr_original_image
echo "$ORIGINAL_AMI_ID"    > /tmp/caa_pr_original_ami
echo "$PR_NUMBER"          > /tmp/caa_pr_number
```

---

## 2. Inspect the PR

Fetch PR metadata via GitHub API (no auth required for public repos):
```bash
PR_API="https://api.github.com/repos/confidential-containers/cloud-api-adaptor/pulls/${PR_NUMBER}"
PR_JSON=$(curl -sL "$PR_API")
PR_TITLE=$(echo "$PR_JSON" | jq -r '.title')
PR_BRANCH=$(echo "$PR_JSON" | jq -r '.head.ref')
PR_REPO=$(echo "$PR_JSON"  | jq -r '.head.repo.clone_url')
PR_SHA=$(echo "$PR_JSON"   | jq -r '.head.sha')
PR_SHORT_SHA=${PR_SHA:0:8}

echo "PR #$PR_NUMBER: $PR_TITLE"
echo "Branch: $PR_BRANCH  SHA: $PR_SHORT_SHA"
echo "Repo:   $PR_REPO"

[[ "$PR_TITLE" != "null" ]] || { echo "ERROR: PR #$PR_NUMBER not found"; exit 1; }
```

Fetch the list of changed files to determine what needs rebuilding:
```bash
FILES_URL="https://api.github.com/repos/confidential-containers/cloud-api-adaptor/pulls/${PR_NUMBER}/files"
CHANGED_FILES=$(curl -sL "$FILES_URL" | jq -r '.[].filename')
echo "Changed files:"
echo "$CHANGED_FILES" | sed 's/^/  /'

# Determine rebuild requirements
NEEDS_CAA_BUILD=false
NEEDS_PODVM_BUILD=false

# CAA daemonset image: any Go code or Dockerfile in cloud-api-adaptor or cloud-providers
echo "$CHANGED_FILES" | grep -qE "^src/cloud-api-adaptor/|^src/cloud-providers/" && NEEDS_CAA_BUILD=true

# PodVM image contains two layers of changes:
#
# 1. mkosi build system — directly changes what goes into the disk image
echo "$CHANGED_FILES" | grep -qE "^src/cloud-api-adaptor/podvm-mkosi/" && NEEDS_PODVM_BUILD=true
#
# 2. Guest binaries — Go packages compiled into the podvm by Dockerfile.podvm_binaries.ubuntu
#    These binaries run INSIDE the VM, not on the worker node:
#      agent-protocol-forwarder  →  pkg/forwarder/, cmd/agent-protocol-forwarder/
#      process-user-data         →  cmd/process-user-data/
#    If any of these change, the binaries-tree must be rebuilt and baked into a new podvm image.
echo "$CHANGED_FILES" | grep -qE "^src/cloud-api-adaptor/pkg/forwarder/\
|^src/cloud-api-adaptor/cmd/agent-protocol-forwarder/\
|^src/cloud-api-adaptor/cmd/process-user-data/\
|^src/cloud-api-adaptor/podvm/Dockerfile\.podvm_binaries" && NEEDS_PODVM_BUILD=true

# Flag overrides
[[ "${FORCE_CAA:-false}"   == "true" ]] && NEEDS_CAA_BUILD=true
[[ "${FORCE_PODVM:-false}" == "true" ]] && NEEDS_PODVM_BUILD=true
[[ "${NO_BUILD:-false}"    == "true" ]] && NEEDS_CAA_BUILD=false && NEEDS_PODVM_BUILD=false

echo ""
echo "Rebuild plan:"
echo "  CAA image:   $([ "$NEEDS_CAA_BUILD"   = true ] && echo YES || echo no)"
echo "  PodVM image: $([ "$NEEDS_PODVM_BUILD" = true ] && echo YES || echo no)"
```

---

## 3. Checkout the PR branch

```bash
cd ~/cloud-api-adaptor

# Record current branch so we can restore it
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
echo "$CURRENT_BRANCH" > /tmp/caa_pr_original_branch

echo "Fetching PR #$PR_NUMBER branch '$PR_BRANCH' ..."
git fetch origin "pull/${PR_NUMBER}/head:pr-${PR_NUMBER}" 2>&1
git checkout "pr-${PR_NUMBER}" 2>&1
echo "Now at: $(git log --oneline -1)"
```

---

## 4. Build CAA container image (if needed)

Skip this step if `NEEDS_CAA_BUILD=false`.

### 4a — ECR login

```bash
aws ecr get-login-password --region "$REGION" | \
    docker login --username AWS --password-stdin "$ECR_REGISTRY"
```

### 4b — Build the CAA image

The CAA image is built from the repo root and includes both `src/cloud-api-adaptor` and `src/cloud-providers`. Use the PR short SHA as the tag:

```bash
PR_CAA_TAG="pr-${PR_NUMBER}-${PR_SHORT_SHA}"
PR_CAA_IMAGE="${ECR_REPO}:${PR_CAA_TAG}"

cd "$CAA_ROOT/podvm-mkosi"
PODVM_DISTRO=ubuntu TEE_PLATFORM=amd make binaries 2>&1

# Build the CAA daemonset image (not the podvm image — that's a separate target)
cd "$CAA_ROOT"
docker buildx build \
    -f Dockerfile \
    -t "${PR_CAA_IMAGE}" \
    --build-arg CLOUD_PROVIDER=aws \
    --progress=plain \
    --load \
    . 2>&1

echo "Built CAA image: $PR_CAA_IMAGE"
echo "$PR_CAA_IMAGE" > /tmp/caa_pr_new_image
```

If there is no top-level `Dockerfile` for the CAA daemonset image, check `src/cloud-api-adaptor/Dockerfile` or build via `make`. Adapt the build command to whatever Makefile target produces the daemonset image.

### 4c — Push to ECR

```bash
docker push "$PR_CAA_IMAGE"
echo "Pushed: $PR_CAA_IMAGE"
```

---

## 5. Build podvm image and create AMI (if needed)

Skip this step if `NEEDS_PODVM_BUILD=false`.

**Always use the debug profile** for PR testing — it includes SSH server, debug tools (strace, vim, curl), and verbose systemd boot. Never use the production profile for PR test images.

### 5a — Inject SSH public key

The debug image build copies `resources/authorized_keys` into `/root/.ssh/authorized_keys` inside the VM (controlled by `mkosi.conf.d/ubuntu-debug-keys.conf`). This is the only way to SSH into the podvm — do it before building:

```bash
SSH_PUBKEY="${SSH_PUBKEY_PATH:-$HOME/.ssh/id_rsa.pub}"
[[ -f "$SSH_PUBKEY" ]] || { echo "WARN: No public key at $SSH_PUBKEY — SSH into podvm will not work"; }
[[ -f "$SSH_PUBKEY" ]] && {
    cp "$SSH_PUBKEY" "$CAA_ROOT/podvm-mkosi/resources/authorized_keys"
    echo "SSH public key injected from $SSH_PUBKEY"
    echo "  → /root/.ssh/authorized_keys inside the podvm"
}
```

To override which key to use: `SSH_PUBKEY_PATH=~/.ssh/other.pub /test-pr 3037`

### 5b — Build the debug podvm image

```bash
cd "$CAA_ROOT/podvm-mkosi"
PODVM_DISTRO=ubuntu make image-debug 2>&1

ls -lh build/system.raw
```

### 5c — How to SSH into the podvm after it launches

The podvm is on a private VPC subnet. SSH via the EKS worker node as a jump host:

```bash
PODVM_IP=$(aws ec2 describe-instances --region "$REGION" \
    --filters "Name=tag:Name,Values=podvm*pr-test*" "Name=instance-state-name,Values=running" \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

WORKER_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

echo "SSH command:"
echo "  ssh -J ubuntu@${WORKER_IP} root@${PODVM_IP}"
```

Or with your key explicitly:
```bash
ssh -i ~/.ssh/id_rsa -J ubuntu@${WORKER_IP} root@${PODVM_IP}
```

Then run the upload flow (equivalent to `/aws-podvm-ami upload`):

```bash
PR_AMI_NAME="podvm-ubuntu-amd64-debug-pr${PR_NUMBER}-${PR_SHORT_SHA}"
BUCKET_NAME="podvm-upstream"   # use existing bucket

# Upload system.raw and register AMI (inline the upload logic)
# ... (reuse the logic from aws-podvm-ami upload subcommand) ...
# Set KEEP_BECAUSE and tag appropriately:
KEEP_BECAUSE="bpradipt - PR #${PR_NUMBER} test"
CAA_VERSION=$(git describe --tags --abbrev=0 2>/dev/null | sed 's/^v//' || echo "dev")

# After successful registration:
echo "$PR_AMI_ID" > /tmp/caa_pr_new_ami
```

Reuse the full upload logic from the `/aws-podvm-ami` skill (S3 upload → import-snapshot → register-image → tag). Pass `--cleanup` to remove the S3 raw after AMI creation.

---

## 6. Deploy to cluster

### 6a — Update CAA daemonset image (if CAA was rebuilt)

```bash
if [[ "$NEEDS_CAA_BUILD" == "true" ]]; then
    PR_CAA_IMAGE=$(cat /tmp/caa_pr_new_image)
    kubectl set image daemonset/cloud-api-adaptor-daemonset \
        cloud-api-adaptor="$PR_CAA_IMAGE" \
        -n confidential-containers-system
    kubectl rollout status daemonset/cloud-api-adaptor-daemonset \
        -n confidential-containers-system --timeout=3m
    echo "CAA daemonset updated to $PR_CAA_IMAGE"
fi
```

### 6b — Update podvm AMI in peer-pods-cm (if podvm was rebuilt)

```bash
if [[ "$NEEDS_PODVM_BUILD" == "true" ]]; then
    PR_AMI_ID=$(cat /tmp/caa_pr_new_ami)
    kubectl patch configmap peer-pods-cm \
        -n confidential-containers-system \
        --type merge \
        -p "{\"data\":{\"PODVM_AMI_ID\":\"${PR_AMI_ID}\"}}"
    # Restart CAA daemonset to pick up the new AMI ID
    kubectl rollout restart daemonset/cloud-api-adaptor-daemonset \
        -n confidential-containers-system
    kubectl rollout status daemonset/cloud-api-adaptor-daemonset \
        -n confidential-containers-system --timeout=3m
    echo "peer-pods-cm updated: PODVM_AMI_ID=$PR_AMI_ID"
fi
```

---

## 7. Test

### 7a — Default smoke test: nginx peer-pod

```bash
# Clean up any previous test pod
kubectl delete deployment nginx-pr-test -n default --ignore-not-found

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pr-test
  namespace: default
  labels:
    pr: "${PR_NUMBER}"
spec:
  selector:
    matchLabels:
      app: nginx-pr-test
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-pr-test
    spec:
      runtimeClassName: kata-remote
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        imagePullPolicy: Always
EOF

echo "Waiting for peer-pod to start (up to 8 min) ..."
kubectl wait --for=condition=Available deployment/nginx-pr-test \
    -n default --timeout=8m || true
kubectl get pods -n default -l app=nginx-pr-test -o wide
```

Verify EC2 VM was created:
```bash
REGION=$(cat /tmp/caa_eks_region)
aws ec2 describe-instances --region "$REGION" \
    --filters "Name=tag:Name,Values=podvm*" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0],InstanceType,PrivateIpAddress]' \
    --output table
```

### 7b — E2E tests (if --test-filter provided)

```bash
if [[ -n "$TEST_FILTER" ]]; then
    cd "$CAA_ROOT"
    export TEST_PROVISION=no
    export CLOUD_PROVIDER=aws
    export TEST_TEARDOWN=no
    export TEST_INSTALL_CAA=no   # already installed
    export TEST_PROVISION_FILE=""
    go test -v -tags=aws \
        -timeout 30m \
        -count=1 \
        -run "$TEST_FILTER" \
        ./test/e2e 2>&1
fi
```

Print test summary:
```
PR #<N> test complete.
  Branch:       <branch>
  CAA image:    <image or "unchanged">
  PodVM AMI:    <ami or "unchanged">
  Test pod:     nginx-pr-test — Running / FAILED

EC2 peer-pod VM: <instance-id> (<type>)
```

---

## 8. Rebuild PodVM only (`--rebuild-podvm`)

Run this when you've iterated on the podvm (changed mkosi config, a guest binary, or postinst scripts) and need a fresh AMI without redoing the full PR test cycle. It assumes Steps 1–3 already ran (the PR branch is checked out and `/tmp/caa_pr_*` state exists).

### Step R1 — Restore context

```bash
export AWS_PROFILE="${AWS_PROFILE:-osc}"
PR_NUMBER=$(cat /tmp/caa_pr_number 2>/dev/null || { echo "ERROR: No PR state found — run /test-pr <N> first"; exit 1; })
REGION=$(cat /tmp/caa_eks_region 2>/dev/null || echo "us-east-2")
KUBECONFIG_FILE=$(cat /tmp/caa_eks_kubeconfig 2>/dev/null)
export KUBECONFIG="$KUBECONFIG_FILE"
CAA_ROOT=~/cloud-api-adaptor/src/cloud-api-adaptor
BUCKET_NAME="podvm-upstream"

# Confirm we're on the PR branch
CURRENT_BRANCH=$(git -C ~/cloud-api-adaptor rev-parse --abbrev-ref HEAD)
PR_SHORT_SHA=$(git -C ~/cloud-api-adaptor rev-parse --short HEAD)
echo "PR #$PR_NUMBER  branch: $CURRENT_BRANCH  SHA: $PR_SHORT_SHA"
```

### Step R2 — Deregister the old PR AMI and delete its snapshot

```bash
OLD_AMI_ID=$(cat /tmp/caa_pr_new_ami 2>/dev/null)
if [[ -n "$OLD_AMI_ID" && "$OLD_AMI_ID" != "null" ]]; then
    OLD_SNAPSHOT=$(aws ec2 describe-images --image-ids "$OLD_AMI_ID" --region "$REGION" \
        --query 'Images[0].BlockDeviceMappings[0].Ebs.SnapshotId' --output text 2>/dev/null)
    aws ec2 deregister-image --image-id "$OLD_AMI_ID" --region "$REGION" 2>/dev/null \
        && echo "Deregistered old PR AMI: $OLD_AMI_ID"
    [[ -n "$OLD_SNAPSHOT" && "$OLD_SNAPSHOT" != "None" ]] && \
        aws ec2 delete-snapshot --snapshot-id "$OLD_SNAPSHOT" --region "$REGION" 2>/dev/null \
        && echo "Deleted old snapshot: $OLD_SNAPSHOT"
else
    echo "No existing PR AMI to deregister — continuing"
fi
```

### Step R3 — Inject SSH key and rebuild debug podvm image

```bash
SSH_PUBKEY="${SSH_PUBKEY_PATH:-$HOME/.ssh/id_rsa.pub}"
[[ -f "$SSH_PUBKEY" ]] && {
    cp "$SSH_PUBKEY" "$CAA_ROOT/podvm-mkosi/resources/authorized_keys"
    echo "SSH key injected: $SSH_PUBKEY"
}

cd "$CAA_ROOT/podvm-mkosi"
PODVM_DISTRO=ubuntu TEE_PLATFORM=amd make binaries 2>&1
PODVM_DISTRO=ubuntu make image-debug 2>&1
ls -lh build/system.raw
```

### Step R4 — Upload raw image, register new AMI, tag it, clean up S3

Run the full upload flow inline (same as Step 5 of the main test flow):

```bash
IMAGE_FILENAME="system-pr${PR_NUMBER}.raw"
PR_AMI_NAME="podvm-ubuntu-amd64-debug-pr${PR_NUMBER}-${PR_SHORT_SHA}"
KEEP_BECAUSE="bpradipt - PR #${PR_NUMBER} rebuild"

TMPDIR=$(mktemp -d)
trap "rm -rf '$TMPDIR'" EXIT
IMAGE_IMPORT_JSON="$TMPDIR/image-import.json"
AMI_REGISTER_JSON="$TMPDIR/register-ami.json"

# Check/fix vmimport policy for podvm-upstream bucket (may have been reset)
EXISTING=$(aws iam get-role-policy --role-name vmimport --policy-name vmimport 2>/dev/null | \
    jq -r '.PolicyDocument.Statement[] | select(.Action[]? | contains("s3:GetObject")) | .Resource[]')
if ! echo "$EXISTING" | grep -q "$BUCKET_NAME"; then
    echo "Adding $BUCKET_NAME to vmimport policy ..."
    MERGED=$(aws iam get-role-policy --role-name vmimport --policy-name vmimport 2>/dev/null | \
        jq '.PolicyDocument' | jq \
        --arg b "arn:aws:s3:::${BUCKET_NAME}" --arg bw "arn:aws:s3:::${BUCKET_NAME}/*" \
        '(.Statement[] | select(.Action[]? | contains("s3:GetObject")) | .Resource) += [$b,$bw]')
    echo "$MERGED" > "$TMPDIR/role-policy.json"
    aws iam put-role-policy --role-name vmimport --policy-name vmimport \
        --policy-document "file://$TMPDIR/role-policy.json"
fi

RAW_IMAGE="$CAA_ROOT/podvm-mkosi/build/system.raw"
echo "Uploading $IMAGE_FILENAME to s3://$BUCKET_NAME/ ..."
aws s3 cp "$RAW_IMAGE" "s3://${BUCKET_NAME}/${IMAGE_FILENAME}" \
    --region "$REGION" --no-progress --expected-size "$(stat -c%s "$RAW_IMAGE")"

cat > "$IMAGE_IMPORT_JSON" <<EOF
{
    "Description": "PR #${PR_NUMBER} podvm rebuild (debug, TEE_PLATFORM=amd)",
    "Format": "RAW",
    "UserBucket": { "S3Bucket": "${BUCKET_NAME}", "S3Key": "${IMAGE_FILENAME}" }
}
EOF

IMPORT_TASK_ID=$(aws ec2 import-snapshot \
    --disk-container "file://$IMAGE_IMPORT_JSON" --output json --region "$REGION" | jq -r '.ImportTaskId')
echo "Import task: $IMPORT_TASK_ID"

ITERATION=0
while [ $ITERATION -lt 120 ]; do
    TASK_JSON=$(aws ec2 describe-import-snapshot-tasks \
        --import-task-ids "$IMPORT_TASK_ID" --output json --region "$REGION")
    STATUS=$(echo "$TASK_JSON" | jq -r '.ImportSnapshotTasks[0].SnapshotTaskDetail.Status')
    PROGRESS=$(echo "$TASK_JSON" | jq -r '.ImportSnapshotTasks[0].SnapshotTaskDetail.Progress // ""')
    echo "[$(date -u +%H:%M:%S)] $STATUS${PROGRESS:+ (${PROGRESS}%)}"
    [[ "$STATUS" == "completed" ]] && break
    [[ "$STATUS" != "active" && "$STATUS" != "pending" ]] && { echo "ERROR: $STATUS"; exit 2; }
    ITERATION=$(( ITERATION + 1 )); sleep 15
done

SNAPSHOT_ID=$(aws ec2 describe-import-snapshot-tasks \
    --import-task-ids "$IMPORT_TASK_ID" --output json --region "$REGION" | \
    jq -r '.ImportSnapshotTasks[0].SnapshotTaskDetail.SnapshotId')
aws ec2 wait snapshot-completed --snapshot-ids "$SNAPSHOT_ID" --region "$REGION"
echo "Snapshot ready: $SNAPSHOT_ID"

cat > "$AMI_REGISTER_JSON" <<EOF
{
    "Architecture": "x86_64",
    "BlockDeviceMappings": [{"DeviceName": "/dev/xvda", "Ebs": {"DeleteOnTermination": true, "SnapshotId": "${SNAPSHOT_ID}"}}],
    "Description": "PR #${PR_NUMBER} podvm rebuild",
    "RootDeviceName": "/dev/xvda", "VirtualizationType": "hvm", "EnaSupport": true, "BootMode": "uefi"
}
EOF

NEW_AMI_ID=$(aws ec2 register-image \
    --name "$PR_AMI_NAME" --cli-input-json "file://$AMI_REGISTER_JSON" \
    --tpm-support v2.0 --output json --region "$REGION" | jq -r '.ImageId')
echo "AMI registered: $NEW_AMI_ID"

aws ec2 create-tags --resources "$NEW_AMI_ID" --region "$REGION" --tags \
    Key=Name,Value="$PR_AMI_NAME" Key=PR,Value="$PR_NUMBER" \
    Key=TEE_PLATFORM,Value=amd Key=Profile,Value=debug \
    Key=keep-because,Value="$KEEP_BECAUSE"

# Save new AMI ID, overwriting the old one
echo "$NEW_AMI_ID" > /tmp/caa_pr_new_ami

# Delete S3 raw object
aws s3 rm "s3://${BUCKET_NAME}/${IMAGE_FILENAME}" --region "$REGION" && echo "S3 object deleted."
```

### Step R5 — Update peer-pods-cm and clean-restart CAA daemonset

Use the clean stop/start pattern (not `rollout restart`) to avoid the hypervisor.sock race:

```bash
echo "Updating peer-pods-cm: PODVM_AMI_ID=$NEW_AMI_ID ..."
kubectl patch configmap peer-pods-cm \
    -n confidential-containers-system \
    --type merge \
    -p "{\"data\":{\"PODVM_AMI_ID\":\"${NEW_AMI_ID}\"}}"

echo "Clean-restarting CAA daemonset ..."
kubectl patch daemonset cloud-api-adaptor-daemonset \
    -n confidential-containers-system \
    --type json \
    -p '[{"op":"replace","path":"/spec/template/spec/nodeSelector","value":{"cloud-api-adaptor-disabled":"true"}}]'
kubectl wait --for=delete pod -n confidential-containers-system \
    -l app=cloud-api-adaptor --timeout=60s 2>/dev/null || true
kubectl patch daemonset cloud-api-adaptor-daemonset \
    -n confidential-containers-system \
    --type json \
    -p '[{"op":"replace","path":"/spec/template/spec/nodeSelector","value":{"node.kubernetes.io/worker":""}}]'
kubectl rollout status daemonset/cloud-api-adaptor-daemonset \
    -n confidential-containers-system --timeout=3m
echo "CAA restarted with new AMI."
```

### Step R6 — Re-run smoke test

Patch the extended resource and redeploy the test pod (same as Step 7):

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl patch node "$NODE" --subresource=status --type=json \
    -p '[{"op":"add","path":"/status/capacity/kata.peerpods.io~1vm","value":"10"}]'

kubectl delete deployment nginx-pr-test -n default --ignore-not-found 2>/dev/null

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pr-test
  namespace: default
  labels:
    pr: "${PR_NUMBER}"
spec:
  selector:
    matchLabels:
      app: nginx-pr-test
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-pr-test
    spec:
      runtimeClassName: kata-remote
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        imagePullPolicy: Always
EOF

echo "Waiting for peer-pod to start (up to 8 min) ..."
kubectl wait --for=condition=Available deployment/nginx-pr-test \
    -n default --timeout=8m || true
echo ""
kubectl get pods -n default -l app=nginx-pr-test -o wide
echo ""
PODVM_IP=$(aws ec2 describe-instances --region "$REGION" \
    --filters "Name=tag:Name,Values=podvm*nginx-pr-test*" \
    "Name=instance-state-name,Values=running" \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
WORKER_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "SSH into podvm:"
echo "  ssh -i ~/.ssh/id_rsa -J ubuntu@${WORKER_IP} root@${PODVM_IP}"
```

---

## 10. Cleanup

Run when `--cleanup` is passed, or invoke separately via `/test-pr <N> --cleanup`.

```bash
PR_NUMBER=$(cat /tmp/caa_pr_number 2>/dev/null)
ORIGINAL_CAA_IMAGE=$(cat /tmp/caa_pr_original_image 2>/dev/null)
ORIGINAL_AMI_ID=$(cat /tmp/caa_pr_original_ami 2>/dev/null)
PR_CAA_IMAGE=$(cat /tmp/caa_pr_new_image 2>/dev/null)
PR_AMI_ID=$(cat /tmp/caa_pr_new_ami 2>/dev/null)
ORIGINAL_BRANCH=$(cat /tmp/caa_pr_original_branch 2>/dev/null)
REGION=$(cat /tmp/caa_eks_region 2>/dev/null)
```

### 8a — Delete test pods
```bash
kubectl delete deployment nginx-pr-test -n default --ignore-not-found
echo "Test pods deleted."
```

### 8b — Revert CAA image to original
```bash
if [[ -n "$ORIGINAL_CAA_IMAGE" ]]; then
    kubectl set image daemonset/cloud-api-adaptor-daemonset \
        cloud-api-adaptor="$ORIGINAL_CAA_IMAGE" \
        -n confidential-containers-system
    kubectl rollout status daemonset/cloud-api-adaptor-daemonset \
        -n confidential-containers-system --timeout=3m
    echo "CAA image reverted to $ORIGINAL_CAA_IMAGE"
fi
```

### 8c — Revert podvm AMI in peer-pods-cm
```bash
if [[ -n "$ORIGINAL_AMI_ID" ]]; then
    kubectl patch configmap peer-pods-cm \
        -n confidential-containers-system \
        --type merge \
        -p "{\"data\":{\"PODVM_AMI_ID\":\"${ORIGINAL_AMI_ID}\"}}"
    kubectl rollout restart daemonset/cloud-api-adaptor-daemonset \
        -n confidential-containers-system
    echo "PODVM_AMI_ID reverted to $ORIGINAL_AMI_ID"
fi
```

### 8d — Delete PR AMI and snapshot
```bash
if [[ -n "$PR_AMI_ID" ]]; then
    SNAPSHOT_ID=$(aws ec2 describe-images --image-ids "$PR_AMI_ID" --region "$REGION" \
        --query 'Images[0].BlockDeviceMappings[0].Ebs.SnapshotId' --output text 2>/dev/null)
    aws ec2 deregister-image --image-id "$PR_AMI_ID" --region "$REGION" 2>/dev/null \
        && echo "AMI $PR_AMI_ID deregistered"
    [[ -n "$SNAPSHOT_ID" && "$SNAPSHOT_ID" != "None" ]] && \
        aws ec2 delete-snapshot --snapshot-id "$SNAPSHOT_ID" --region "$REGION" 2>/dev/null \
        && echo "Snapshot $SNAPSHOT_ID deleted"
fi
```

### 8e — Delete PR ECR image tag
```bash
if [[ -n "$PR_CAA_IMAGE" ]]; then
    PR_CAA_TAG="${PR_CAA_IMAGE##*:}"
    aws ecr batch-delete-image \
        --repository-name cloud-api-adaptor \
        --region "$REGION" \
        --image-ids imageTag="$PR_CAA_TAG" 2>/dev/null \
        && echo "ECR image tag $PR_CAA_TAG deleted"
fi
```

### 8f — Restore git branch
```bash
if [[ -n "$ORIGINAL_BRANCH" ]]; then
    cd ~/cloud-api-adaptor
    git checkout "$ORIGINAL_BRANCH" 2>&1
    git branch -D "pr-${PR_NUMBER}" 2>/dev/null || true
    echo "Git branch restored to $ORIGINAL_BRANCH"
fi

# Clean up temp state
rm -f /tmp/caa_pr_original_image /tmp/caa_pr_original_ami /tmp/caa_pr_new_image \
      /tmp/caa_pr_new_ami /tmp/caa_pr_number /tmp/caa_pr_original_branch
echo "Cleanup complete."
```

</process>

<known_pitfalls>
## 1. GitHub API rate limit
Unauthenticated requests to `api.github.com` are limited to 60/hour per IP. If hitting limits, set `GH_TOKEN` env var and add `-H "Authorization: Bearer $GH_TOKEN"` to curl calls.

## 2. CAA Dockerfile location
There is no single top-level `Dockerfile` for the CAA daemonset image — the build is done via `make` targets in `src/cloud-api-adaptor/Makefile`. Use `RELEASE_BUILD=true BUILTIN_CLOUD_PROVIDERS=aws make cloud-api-adaptor` to build the binary, then wrap it in a minimal container image. Or check if the repo's Makefile has a `image` target that builds a daemonset container.

## 3. ECR image pull permissions from EKS
The EKS node IAM role needs `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, and `ecr:BatchCheckLayerAvailability` on the ECR repo. These are typically present via the `AmazonEC2ContainerRegistryReadOnly` managed policy attached to the node role. If image pulls fail, check the node group IAM role.

## 4. podvm rebuild only needed when mkosi.postinst or resources/ change
CAA Go code changes do NOT require a new podvm image — the podvm image only needs rebuilding when `src/cloud-api-adaptor/podvm-mkosi/` files change. The diff detection in Step 2 handles this automatically.

## 5. kubectl rollout restart vs set image
`kubectl set image` triggers an immediate rolling restart. `kubectl rollout restart` forces a restart without an image change (needed when only the ConfigMap changes). Both are used as appropriate.

## 9. Correct container name for `kubectl set image`
The CAA daemonset container is named `cloud-api-adaptor-con`, not `cloud-api-adaptor`. Always resolve the name dynamically:
```bash
CONTAINER_NAME=$(kubectl get daemonset cloud-api-adaptor-daemonset \
    -n confidential-containers-system \
    -o jsonpath='{.spec.template.spec.containers[0].name}')
kubectl set image daemonset/cloud-api-adaptor-daemonset \
    "${CONTAINER_NAME}=${PR_CAA_IMAGE}" -n confidential-containers-system
```

## 10. `kata.peerpods.io/vm` extended resource cleared by AL2023 kubelet
The CAA's `AdvertiseExtendedResources()` patches the node status, but the AL2023 managed kubelet periodically overwrites it. The resource appears briefly then disappears. Workaround: manually patch the node immediately before deploying the test pod:
```bash
kubectl patch node "$NODE" --subresource=status --type=json \
    -p '[{"op":"add","path":"/status/capacity/kata.peerpods.io~1vm","value":"10"}]'
```

## 11. CAA socket race during rolling restart with shared hostPath
`/run/peerpod` is a hostPath mounted by all CAA pods. During `rollout restart`, the new pod creates `hypervisor.sock`, then the OLD pod's graceful shutdown removes it (the shutdown calls `os.RemoveAll(socketPath)`). The socket disappears and containerd shim gets "no such file or directory".
Fix: instead of `rollout restart`, do a clean stop/start:
```bash
# Stop all CAA pods first
kubectl patch daemonset cloud-api-adaptor-daemonset -n confidential-containers-system \
    --type json -p '[{"op":"replace","path":"/spec/template/spec/nodeSelector","value":{"cloud-api-adaptor-disabled":"true"}}]'
kubectl wait --for=delete pod -n confidential-containers-system -l app=cloud-api-adaptor --timeout=60s
# Then start fresh
kubectl patch daemonset cloud-api-adaptor-daemonset -n confidential-containers-system \
    --type json -p '[{"op":"replace","path":"/spec/template/spec/nodeSelector","value":{"node.kubernetes.io/worker":""}}]'
```

## 12. Probe port 8000 already in use during rolling restart
With `hostNetwork: true`, the probe server on port 8000 is on the host network. If the old pod is still running when the new pod starts (rolling restart), the new pod fails to bind port 8000. This blocks the startup probe but NOT the main CAA server. Resolved by the clean stop/start approach in pitfall #11.

## 6. PR branch may be from a fork
If `PR_REPO` differs from the upstream repo, `git fetch` must target the fork's remote URL, not `origin`. The `pull/<N>/head` ref works directly via the upstream remote regardless of fork.

## 7. go.sum mismatch after branch switch
After checking out the PR branch, `go mod tidy` may be needed if `go.mod` changed. Run `cd src/cloud-api-adaptor && go mod tidy && cd ../cloud-providers && go mod tidy` before building.
</known_pitfalls>

<success_criteria>
**inspect:** PR title, branch, and rebuild plan printed cleanly
**build:** new ECR image tag pushed; new AMI registered (when applicable)
**deploy:** `kubectl get daemonset cloud-api-adaptor-daemonset` shows the new image; `peer-pods-cm` shows the new AMI ID
**test:** `nginx-pr-test` pod reaches Running; EC2 table shows a new `podvm-*` instance
**cleanup:** daemonset image back to original; PODVM_AMI_ID back to original; PR AMI deregistered; git on original branch
</success_criteria>
