---
name: aws-eks-caa
description: Set up Cloud API Adaptor (CAA) on AWS EKS — creates the cluster, configures networking, installs the CAA Helm chart with a custom podvm AMI, and runs a test workload. Supports teardown. AWS credentials must be in the environment.
argument-hint: "setup [--ami <AMI_ID>] [--region <REGION>] [--cluster-name <NAME>] [--instance-type <TYPE>] [--non-cvm] | test | teardown [--cluster-name <NAME>]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Automate the full lifecycle of a CAA peer-pods deployment on AWS EKS:

- **setup** — create EKS cluster, configure networking, deploy CAA via Helm with a custom podvm AMI
- **test** — deploy a sample nginx pod via kata-remote runtimeClass and verify a peer-pod VM appears in EC2
- **teardown** — delete the nginx workload and EKS cluster

AWS credentials are assumed to be in the environment (`AWS_PROFILE`, instance role, or env vars). No credential prompting.

Default values (all overridable via flags):
- Region: `us-east-2`
- Instance type for podvm: `m6a.large` (AMD SEV-SNP capable)
- EKS node type: `m5.xlarge`
- CVM mode: enabled (`DISABLECVM=false`)
</objective>

<repo_layout>
```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/
        └── install/
            └── charts/
                └── peerpods/         ← Helm chart root (CHART_DIR)
                    ├── Chart.yaml
                    ├── values.yaml
                    └── providers/
                        ├── aws.yaml              ← provider values template
                        └── aws-secrets.yaml.template
```
</repo_layout>

<process>

## 0. Parse Arguments

Parse `$ARGUMENTS`:
- First positional arg: subcommand (`setup`, `test`, `teardown`)
- `--ami <id>` — podvm AMI ID (required for setup; no default — must be supplied)
- `--region <region>` — AWS region (default: `us-east-2`)
- `--cluster-name <name>` — EKS cluster name (default: `caa-<timestamp>`)
- `--instance-type <type>` — podvm EC2 instance type (default: `m6a.large` for AMD SEV-SNP)
- `--non-cvm` — set `DISABLECVM=true` and switch podvm instance to `t3.large` (non-confidential testing)

If no subcommand given, ask:
```
Which phase do you want to run?
  1) setup    — create EKS cluster and deploy CAA
  2) test     — run an nginx peer-pod and verify the EC2 VM
  3) teardown — delete workloads and the EKS cluster
```

---

## 1. Locate the Repo and Chart

```bash
CAA_ROOT=~/cloud-api-adaptor/src/cloud-api-adaptor
CHART_DIR="$CAA_ROOT/install/charts/peerpods"
ls "$CHART_DIR/Chart.yaml" 2>/dev/null || { echo "ERROR: Helm chart not found at $CHART_DIR"; exit 1; }
```

---

## 2. Preflight Check (all subcommands)

```bash
# AWS credentials
AWS_ARN=$(aws sts get-caller-identity 2>/dev/null | jq -r '.Arn')
[[ -n "$AWS_ARN" ]] || { echo "BLOCK: AWS credentials not found"; exit 1; }

# Clock skew (EC2 SigV4 requires <5 min drift)
AWS_TIME=$(curl -sI https://aws.amazon.com 2>/dev/null | grep -i '^date:' | sed 's/date: //i' | tr -d '\r')
AWS_EPOCH=$(date -d "$AWS_TIME" +%s 2>/dev/null || echo 0)
LOCAL_EPOCH=$(date -u +%s)
SKEW=$(( AWS_EPOCH - LOCAL_EPOCH ))
SKEW_ABS=${SKEW#-}
if [ "${SKEW_ABS:-0}" -gt 240 ]; then
    echo "WARN: Clock skew ${SKEW}s — syncing from AWS ..."
    sudo date -s "$AWS_TIME"
fi

# Required tools
for tool in aws eksctl kubectl helm jq; do
    command -v "$tool" &>/dev/null && echo "  $tool: ok" || echo "  $tool: MISSING"
done
```

**Install eksctl if missing:**
```bash
command -v eksctl &>/dev/null || (
    EKSCTL_VERSION=$(curl -sL https://api.github.com/repos/eksctl-io/eksctl/releases/latest | jq -r '.tag_name')
    curl -sLO "https://github.com/eksctl-io/eksctl/releases/download/${EKSCTL_VERSION}/eksctl_Linux_amd64.tar.gz"
    tar -xzf eksctl_Linux_amd64.tar.gz
    sudo install -m 0755 eksctl /usr/local/bin/eksctl
    rm -f eksctl eksctl_Linux_amd64.tar.gz
    echo "eksctl installed: $(eksctl version)"
)
```

Block if aws, kubectl, helm, or jq are missing after install attempt.

---

## SUBCOMMAND: setup

### Step 1 — Resolve parameters

```bash
REGION="${REGION_ARG:-us-east-2}"
PODVM_AMI_ID="${AMI_ARG:?ERROR: --ami <AMI_ID> is required}"
CLUSTER_NAME="${CLUSTER_NAME_ARG:-caa-$(date '+%Y%m%d%H%M%S')}"
KUBECONFIG_FILE="${HOME}/${CLUSTER_NAME}-kubeconfig"
EKS_NODE_TYPE="m5.xlarge"
CAA_VERSION="0.21.0"
CAA_IMAGE="quay.io/confidential-containers/cloud-api-adaptor"
CAA_TAG="v${CAA_VERSION}-amd64"

if [[ "${NON_CVM:-false}" == "true" ]]; then
    PODVM_INSTANCE_TYPE="t3.large"
    DISABLECVM="true"
else
    PODVM_INSTANCE_TYPE="${INSTANCE_TYPE_ARG:-m6a.large}"
    DISABLECVM="false"
fi

echo "Configuration:"
echo "  Cluster:          $CLUSTER_NAME"
echo "  Region:           $REGION"
echo "  PodVM AMI:        $PODVM_AMI_ID"
echo "  PodVM type:       $PODVM_INSTANCE_TYPE"
echo "  CVM mode:         $([ "$DISABLECVM" = "false" ] && echo enabled || echo disabled)"
echo "  CAA image:        $CAA_IMAGE:$CAA_TAG"
echo "  Kubeconfig:       $KUBECONFIG_FILE"

# Persist cluster name for use by test/teardown subcommands
echo "$CLUSTER_NAME" > /tmp/caa_eks_cluster_name
echo "$REGION" > /tmp/caa_eks_region
echo "$KUBECONFIG_FILE" > /tmp/caa_eks_kubeconfig
```

Verify the AMI exists in the target region before proceeding:
```bash
aws ec2 describe-images --image-ids "$PODVM_AMI_ID" --region "$REGION" \
    --query 'Images[0].{ID:ImageId,State:State,Name:Name}' --output table 2>/dev/null \
    || { echo "ERROR: AMI $PODVM_AMI_ID not found in $REGION"; exit 1; }
```

### Step 2 — Create EKS cluster

```bash
echo "Creating EKS cluster $CLUSTER_NAME (this takes ~15 minutes) ..."
eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$REGION" \
    --node-type "$EKS_NODE_TYPE" \
    --node-ami-family AmazonLinux2023 \
    --nodes 1 \
    --nodes-min 0 \
    --nodes-max 2 \
    --node-private-networking \
    --kubeconfig "$KUBECONFIG_FILE"

export KUBECONFIG="$KUBECONFIG_FILE"
```

Label the worker nodes:
```bash
for NODE in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
    kubectl label node "$NODE" node.kubernetes.io/worker= --overwrite
done
echo "Worker nodes labelled."
```

Verify:
```bash
kubectl get nodes
```

Expected: at least 1 node in `Ready` state.

### Step 2.5 — Install fuse on worker nodes (required for nydus snapshotter)

`kata-remote` uses the `nydus-for-kata-tee` snapshotter which requires `mount.fuse` on the host. AL2023 nodes don't include it. Deploy a privileged DaemonSet to install it before any peer-pod workloads run:

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fuse-installer
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fuse-installer
  template:
    metadata:
      labels:
        name: fuse-installer
    spec:
      hostPID: true
      tolerations:
      - operator: Exists
      initContainers:
      - name: install-fuse
        image: busybox
        command: ["/bin/sh", "-c", "chroot /host /bin/bash -c 'dnf install -y fuse fuse3 2>&1 && echo DONE'"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: host-root
          mountPath: /host
      containers:
      - name: pause
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.9
      volumes:
      - name: host-root
        hostPath:
          path: /
EOF

# Wait for the initContainer to finish on all nodes
echo "Waiting for fuse install to complete on all nodes ..."
kubectl wait --for=jsonpath='{.status.numberReady}'=1 daemonset/fuse-installer \
    -n kube-system --timeout=5m
kubectl logs -n kube-system -l name=fuse-installer -c install-fuse --tail=5 2>/dev/null || true
echo "fuse installed on all worker nodes."
```

### Step 3 — Open required network ports

```bash
echo "Configuring security group ingress rules ..."

EKS_VPC_ID=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" \
    --query "cluster.resourcesVpcConfig.vpcId" --output text)

EKS_CLUSTER_SG=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" \
    --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

EKS_VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "$EKS_VPC_ID" --region "$REGION" \
    --query 'Vpcs[0].CidrBlock' --output text)

echo "  VPC:              $EKS_VPC_ID"
echo "  Security group:   $EKS_CLUSTER_SG"
echo "  VPC CIDR:         $EKS_VPC_CIDR"

# agent-protocol-forwarder (TCP 15150)
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --region "$REGION" \
    --protocol tcp --port 15150 --cidr "$EKS_VPC_CIDR" 2>/dev/null \
    && echo "  TCP 15150 (agent-protocol-forwarder): added" \
    || echo "  TCP 15150: already exists"

# VXLAN (TCP + UDP 9000)
aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --region "$REGION" \
    --protocol tcp --port 9000 --cidr "$EKS_VPC_CIDR" 2>/dev/null \
    && echo "  TCP 9000 (VXLAN): added" || echo "  TCP 9000: already exists"

aws ec2 authorize-security-group-ingress --group-id "$EKS_CLUSTER_SG" --region "$REGION" \
    --protocol udp --port 9000 --cidr "$EKS_VPC_CIDR" 2>/dev/null \
    && echo "  UDP 9000 (VXLAN): added" || echo "  UDP 9000: already exists"

# Persist networking details for use in aws.yaml
echo "$EKS_CLUSTER_SG" > /tmp/caa_eks_sg_id
echo "$EKS_VPC_ID" > /tmp/caa_eks_vpc_id
echo "$EKS_VPC_CIDR" > /tmp/caa_eks_vpc_cidr
```

Retrieve a subnet ID in the VPC for the podvm launch:
```bash
# Use the first private subnet associated with the cluster node group
SUBNET_ID=$(aws ec2 describe-subnets --region "$REGION" \
    --filters "Name=vpc-id,Values=$EKS_VPC_ID" "Name=mapPublicIpOnLaunch,Values=false" \
    --query 'Subnets[0].SubnetId' --output text)

# Fall back to any subnet in the VPC if no private subnet found
[[ "$SUBNET_ID" == "None" || -z "$SUBNET_ID" ]] && \
    SUBNET_ID=$(aws ec2 describe-subnets --region "$REGION" \
        --filters "Name=vpc-id,Values=$EKS_VPC_ID" \
        --query 'Subnets[0].SubnetId' --output text)

echo "  Subnet for podvms: $SUBNET_ID"
echo "$SUBNET_ID" > /tmp/caa_eks_subnet_id
```

### Step 4 — Write providers/aws.yaml

Write the provider configuration into the chart directory (this file is gitignored and safe to populate):

```bash
CHART_DIR="$CAA_ROOT/install/charts/peerpods"

cat > "$CHART_DIR/providers/aws.yaml" <<EOF
provider: aws

image:
  name: "${CAA_IMAGE}"
  tag: "${CAA_TAG}"

providerConfigs:
  aws:
    AWS_REGION: "${REGION}"
    PODVM_AMI_ID: "${PODVM_AMI_ID}"
    PODVM_INSTANCE_TYPE: "${PODVM_INSTANCE_TYPE}"
    DISABLECVM: "${DISABLECVM}"
    AWS_SG_IDS: "${EKS_CLUSTER_SG}"
    AWS_SUBNET_ID: "${SUBNET_ID}"
    VXLAN_PORT: "9000"
EOF

echo "providers/aws.yaml written."
cat "$CHART_DIR/providers/aws.yaml"
```

### Step 5 — Create namespace and credentials secret

```bash
export KUBECONFIG="$KUBECONFIG_FILE"

kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: confidential-containers-system
  labels:
    app.kubernetes.io/managed-by: Helm
  annotations:
    meta.helm.sh/release-name: peerpods
    meta.helm.sh/release-namespace: confidential-containers-system
EOF

# Get current AWS credentials from environment / profile
AWS_ACCESS_KEY_ID_VAL=$(aws configure get aws_access_key_id 2>/dev/null || echo "${AWS_ACCESS_KEY_ID:-}")
AWS_SECRET_ACCESS_KEY_VAL=$(aws configure get aws_secret_access_key 2>/dev/null || echo "${AWS_SECRET_ACCESS_KEY:-}")
AWS_SESSION_TOKEN_VAL=$(aws configure get aws_session_token 2>/dev/null || echo "${AWS_SESSION_TOKEN:-}")

kubectl create secret generic my-provider-creds \
    -n confidential-containers-system \
    --from-literal=AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID_VAL}" \
    --from-literal=AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY_VAL}" \
    $([ -n "$AWS_SESSION_TOKEN_VAL" ] && echo "--from-literal=AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN_VAL}") \
    --dry-run=client -o yaml | kubectl apply -f -

echo "Credentials secret created."
```

### Step 5.5 — Install cert-manager (required by webhook subchart)

The webhook subchart requires cert-manager CRDs to exist before `helm install` runs or it will fail at the resource-mapping stage.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=Available deployment -n cert-manager --all --timeout=5m
```

### Step 6 — Install the CAA Helm chart

Install with all subcharts enabled: the webhook injects peer-pod resources into pod specs, and the peerpod-ctrl garbage-collects dangling cloud resources. cert-manager (installed in step 5.5) is required by the webhook subchart.

```bash
cd "$CHART_DIR"
helm install peerpods . \
    -f providers/aws.yaml \
    --set secrets.mode=reference \
    --set secrets.existingSecretName=my-provider-creds \
    --dependency-update \
    -n confidential-containers-system \
    --wait \
    --timeout 10m

echo "Helm install complete."
```

### Step 7 — Verify deployment

```bash
export KUBECONFIG="$KUBECONFIG_FILE"

echo "Checking runtimeclass ..."
kubectl get runtimeclass

echo ""
echo "Checking CAA pods ..."
kubectl get pods -n confidential-containers-system

echo ""
echo "Checking kata-agent on nodes ..."
kubectl get nodes -o wide
```

Expected: `kata-remote` runtimeClass present, CAA daemonset pod Running on each worker node.

Print setup summary:
```
Setup complete.
  Cluster:     $CLUSTER_NAME
  Region:      $REGION
  Kubeconfig:  $KUBECONFIG_FILE
  AMI:         $PODVM_AMI_ID
  PodVM type:  $PODVM_INSTANCE_TYPE
  CVM:         $([ "$DISABLECVM" = "false" ] && echo "enabled (AMD SEV-SNP)" || echo "disabled")

Next steps:
  /aws-eks-caa test                         — run a peer-pod workload
  /aws-eks-caa teardown                     — delete everything when done
```

---

## SUBCOMMAND: test

### Step 1 — Restore context

```bash
CLUSTER_NAME=$(cat /tmp/caa_eks_cluster_name 2>/dev/null || echo "")
REGION=$(cat /tmp/caa_eks_region 2>/dev/null || echo "us-east-2")
KUBECONFIG_FILE=$(cat /tmp/caa_eks_kubeconfig 2>/dev/null || echo "${HOME}/${CLUSTER_NAME}-kubeconfig")
export KUBECONFIG="$KUBECONFIG_FILE"

[[ -f "$KUBECONFIG_FILE" ]] || { echo "ERROR: kubeconfig not found at $KUBECONFIG_FILE — run setup first"; exit 1; }
kubectl get runtimeclass kata-remote &>/dev/null || { echo "ERROR: kata-remote runtimeClass not found — run setup first"; exit 1; }
```

### Step 2 — Deploy nginx peer-pod

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-peerpod
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx-peerpod
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-peerpod
    spec:
      runtimeClassName: kata-remote
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        imagePullPolicy: Always
EOF

echo "Deployment created. Waiting for pod to be scheduled ..."
kubectl wait --for=condition=Available deployment/nginx-peerpod -n default --timeout=5m || true
kubectl get pods -n default -l app=nginx-peerpod -o wide
```

### Step 3 — Verify EC2 peer-pod VM appeared

```bash
echo "Checking for peer-pod EC2 instance ..."
sleep 15  # allow a moment for VM creation to register
aws ec2 describe-instances --region "$REGION" \
    --filters "Name=tag:Name,Values=podvm*" "Name=instance-state-name,Values=running,pending" \
    --query 'Reservations[*].Instances[*].[InstanceId, Tags[?Key==`Name`].Value|[0], State.Name, InstanceType]' \
    --output table
```

Print test summary:
```
Test complete.
  Pod:     nginx-peerpod (kata-remote runtimeClass)
  Check the EC2 table above for the peer-pod VM instance.

To clean up just the test workload:
  kubectl delete deployment nginx-peerpod -n default
```

---

## SUBCOMMAND: teardown

### Step 1 — Restore context

```bash
CLUSTER_NAME=$(cat /tmp/caa_eks_cluster_name 2>/dev/null || echo "")
REGION=$(cat /tmp/caa_eks_region 2>/dev/null || echo "us-east-2")
KUBECONFIG_FILE=$(cat /tmp/caa_eks_kubeconfig 2>/dev/null || echo "${HOME}/${CLUSTER_NAME}-kubeconfig")
export KUBECONFIG="$KUBECONFIG_FILE"

[[ -n "$CLUSTER_NAME" ]] || { echo "ERROR: No cluster name found. Did setup run?"; exit 1; }
echo "Will delete cluster: $CLUSTER_NAME in $REGION"
```

### Step 2 — Delete test workloads

```bash
echo "Deleting peer-pod workloads ..."
kubectl delete deployment nginx-peerpod -n default --ignore-not-found

echo "Waiting for peer-pod EC2 VMs to terminate ..."
sleep 30
aws ec2 describe-instances --region "$REGION" \
    --filters "Name=tag:Name,Values=podvm*" "Name=instance-state-name,Values=running,pending,stopping" \
    --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' --output table
```

### Step 3 — Uninstall Helm chart

```bash
helm uninstall peerpods -n confidential-containers-system 2>/dev/null || true
kubectl delete namespace confidential-containers-system --ignore-not-found --wait --timeout=60s
```

### Step 4 — Delete EKS cluster

```bash
echo "Deleting EKS cluster $CLUSTER_NAME (this takes ~10 minutes) ..."
eksctl delete cluster --name "$CLUSTER_NAME" --region "$REGION"
echo "Cluster deleted."

# Clean up temp files
rm -f /tmp/caa_eks_cluster_name /tmp/caa_eks_region /tmp/caa_eks_kubeconfig \
      /tmp/caa_eks_sg_id /tmp/caa_eks_vpc_id /tmp/caa_eks_subnet_id
echo "Teardown complete."
```

</process>

<known_pitfalls>
## 1. eksctl not installed
Install via GitHub releases — do not use `apt` (package is often stale). See preflight step.

## 0. Ubuntu2204 AMI family not found in SSM
`eksctl` with `Ubuntu2204` fails with `ParameterNotFound` for newer k8s versions (1.34). Use `AmazonLinux2023` instead — it has full SSM coverage across all k8s versions. Drop the `--node-ami-family Ubuntu2204` flag; AL2023 is the right default for EKS.

## 0b. Stale CloudFormation stack from interrupted eksctl run
If `eksctl create cluster` fails or is interrupted, it leaves a CF stack behind that blocks re-runs with the same cluster name. Fix: generate a new cluster name (timestamp suffix), OR: disable termination protection and delete the stack manually:
```bash
aws cloudformation update-termination-protection --no-enable-termination-protection \
    --stack-name "eksctl-<old-name>-cluster" --region $REGION
aws cloudformation delete-stack --stack-name "eksctl-<old-name>-cluster" --region $REGION
aws cloudformation wait stack-delete-complete --stack-name "eksctl-<old-name>-cluster" --region $REGION
```

## 2. Clock skew breaks EC2 API calls
EC2 SigV4 requires system clock within 5 min of AWS time. Sync from `aws.amazon.com` Date header before any EC2 call. See preflight check.

## 3. vmimport role policy scope
If the vmimport IAM role already exists from a prior run, its inline policy may only cover a different S3 bucket. The `/aws-podvm-ami upload` skill handles merging new buckets in; no action needed here.

## 4. Security group rules already exist
`authorize-security-group-ingress` returns an error if the rule already exists. The setup step uses `2>/dev/null || echo "already exists"` to suppress these non-fatal errors.

## 5. Private subnet detection
`--node-private-networking` in eksctl puts nodes in private subnets. The subnet lookup uses `mapPublicIpOnLaunch=false` to find private subnets. If the cluster uses public subnets instead, fall back to any subnet in the VPC.

## 6. CAA_TAG must match a published image
Check before setting: `curl -sI "https://quay.io/v2/confidential-containers/cloud-api-adaptor/manifests/v${CAA_VERSION}-amd64" -o /dev/null -w "%{http_code}"`. Use `v0.20.0-amd64` if the target version isn't published yet (`404` response).

## 9. AL2023 worker nodes missing `mount.fuse` — nydus snapshotter fails
`kata-remote` uses the `nydus-for-kata-tee` snapshotter on the host node to set up the container bundle (even for `image_guest_pull` where the actual pull happens in the podvm). AL2023 doesn't ship `fuse` by default so `mount.fuse` is missing and `CreateContainerError` loops forever.

Fix: install `fuse` on each worker node via a privileged DaemonSet using `chroot /host`:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fuse-installer
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fuse-installer
  template:
    metadata:
      labels:
        name: fuse-installer
    spec:
      hostPID: true
      tolerations:
      - operator: Exists
      initContainers:
      - name: install-fuse
        image: busybox
        command: ["/bin/sh", "-c", "chroot /host /bin/bash -c 'dnf install -y fuse fuse3 2>&1 && echo DONE'"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: host-root
          mountPath: /host
      containers:
      - name: pause
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.9
      volumes:
      - name: host-root
        hostPath:
          path: /
```
This must be deployed and the initContainer must complete (`DONE` in logs) before peer-pod workloads are scheduled. The DaemonSet can remain running — the initContainer only runs once per node.

## 7. Helm chart dependency-update requires network access
`--dependency-update` downloads the kata-deploy sub-chart on first install. Requires access to `ghcr.io`. If it fails, run `helm dependency update` manually in `CHART_DIR` first.

## 8. Credentials secret — AWS_SESSION_TOKEN
If the profile uses temporary credentials (STS assumed role), `AWS_SESSION_TOKEN` must also be included in the secret or CAA will get `ExpiredToken` errors from EC2. The setup step reads it from the profile config automatically.
</known_pitfalls>

<success_criteria>
**setup:** `kubectl get runtimeclass` shows `kata-remote`; CAA daemonset pod is `Running` in `confidential-containers-system`
**test:** `kubectl get pods -n default` shows the nginx pod `Running`; `aws ec2 describe-instances` shows a `podvm-*` instance in `running` state
**teardown:** EKS cluster deleted; no `podvm-*` EC2 instances remain in `running` state
</success_criteria>
