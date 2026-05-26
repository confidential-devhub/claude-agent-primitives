---
name: aws-podvm-ami
description: Build a debug Ubuntu podvm mkosi image with TEE_PLATFORM=amd for AWS, then upload the raw image to S3 and register it as an AMI. AWS credentials are assumed to be available in the environment.
argument-hint: "build | upload --bucket <S3_BUCKET> [--region <REGION>] [--name <AMI_NAME>] [--cleanup] | all --bucket <S3_BUCKET> [--region <REGION>] [--name <AMI_NAME>] [--cleanup]"
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
Automate the full workflow for building a debug Ubuntu podvm mkosi image with TEE_PLATFORM=amd (AMD SEV-SNP attestation) and uploading it as an AWS AMI.

Subcommands:
- **build** — build binaries (TEE_PLATFORM=amd) + debug mkosi image; produces `build/system.raw`
- **upload** — upload an existing `build/system.raw` to S3 and register as an AMI
- **all** — build then upload in one sequence

AWS credentials are assumed already configured in the environment (AWS_PROFILE, instance role, etc). No credential prompting.
</objective>

<repo_layout>
```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/          ← CAA_ROOT
        ├── Makefile.defaults       ← TEE_PLATFORM, ARCH defaults
        ├── podvm-mkosi/
        │   ├── Makefile            ← make binaries / make image-debug
        │   └── build/
        │       ├── system.raw      ← raw image (input for AMI upload)
        │       └── podvm-ubuntu-amd64.qcow2  ← qcow2 (not used for AMI)
        └── aws/
            └── raw-to-ami.sh       ← reference upload script (see notes)
```
</repo_layout>

<raw_to_ami_notes>
The existing `aws/raw-to-ami.sh` is a reference implementation. This skill does NOT call it directly — it runs equivalent logic inline with these improvements:

1. **Idempotent bucket creation** — check with `aws s3api head-bucket` before creating; skip if it already exists
2. **Idempotent IAM role** — check with `aws iam get-role` before creating; skip role + policy creation if already present
3. **Single poll API call** — combine status + message from one `describe-import-snapshot-tasks` call
4. **TMPDIR trap** — `trap "rm -rf $TMPDIR" EXIT` so JSON temp files are always cleaned up
5. **Format extraction fix** — use `${IMAGE_NAME##*.}` not `${IMAGE_NAME#*.}` to handle multi-dot filenames like `podvm-ubuntu-amd64.raw` correctly
6. **Validate SNAPSHOT_ID** — check it is non-empty before calling register-image
7. **AMI tags** — tag with TEE_PLATFORM, debug=true, build-date, git-commit-sha
8. **Optional S3 cleanup** — if `--cleanup` is passed, delete the raw object from S3 after successful AMI registration
</raw_to_ami_notes>

<process>

## 0. Parse Arguments

Parse `$ARGUMENTS`:
- First positional arg: subcommand (`build`, `upload`, `all`)
- `--bucket <name>` — S3 bucket name (required for `upload` and `all`)
- `--region <region>` — AWS region; falls back to `$AWS_REGION` env var
- `--name <ami-name>` — custom AMI name (default: `podvm-ubuntu-amd64-debug-<git-sha>`)
- `--cleanup` — delete the S3 raw object after successful AMI registration

If no subcommand given, ask:
```
Which phase do you want to run?
  1) build  — build binaries + debug mkosi image (TEE_PLATFORM=amd)
  2) upload — upload existing build/system.raw to S3 and register as AMI
  3) all    — build then upload
```

---

## 1. Locate the Repo

```bash
ls ~/cloud-api-adaptor/src/cloud-api-adaptor/Makefile 2>/dev/null && echo "found"
```

If not found, ask the user for the path. Set `CAA_ROOT` to the resolved path. All subsequent commands use `CAA_ROOT`.

---

## 2. Preflight Check

Run before any subcommand:

```bash
# Disk space
df -BG --output=avail "$CAA_ROOT" | tail -1 | tr -d 'G '
df -BG --output=avail /var/lib/docker 2>/dev/null | tail -1 | tr -d 'G '

# RAM
free -g | awk '/^Mem:/ {print $2, $7}'

# CPU
nproc

# Docker
docker info &>/dev/null && echo "docker_ok" || echo "docker_missing"
```

For **build** subcommand, minimum requirements:
- 20 GB free disk on the repo/Docker partition (Docker build layers ~4-6 GB, system.raw ~1.5 GB)
- Docker must be running

For **upload** subcommand:
```bash
# Verify AWS credentials
aws sts get-caller-identity 2>/dev/null | jq -r '.Arn' || echo "aws_creds_missing"

# Verify required tools
command -v jq &>/dev/null && echo "jq_ok" || echo "jq_missing"
command -v aws &>/dev/null && echo "aws_ok" || echo "aws_missing"
```

Block if: docker missing (build), AWS creds missing (upload), jq/aws-cli missing (upload).

**Clock skew check (upload only):** EC2 SigV4 requires the system clock to be within 5 minutes of AWS time. S3 may succeed even with larger drift (uploads span many requests), but EC2 will fail. Always verify before starting:
```bash
LOCAL_TIME=$(date -u +%s)
AWS_TIME=$(curl -sI https://aws.amazon.com 2>/dev/null | grep -i '^date:' | sed 's/date: //i' | tr -d '\r')
AWS_EPOCH=$(date -d "$AWS_TIME" +%s 2>/dev/null || echo 0)
SKEW=$(( AWS_EPOCH - LOCAL_TIME ))
SKEW_ABS=${SKEW#-}
if [ "${SKEW_ABS:-0}" -gt 240 ]; then
    echo "WARN: Clock skew ${SKEW}s — syncing from AWS ..."
    sudo date -s "$AWS_TIME"
fi
```

Print a preflight summary:
```
Preflight check:
  Disk (repo):   <N> GB free   [OK / BLOCK]
  Docker:        ok/missing    [OK / BLOCK]
  AWS identity:  <ARN>         [OK / BLOCK]    (upload only)
  Tools (jq):    ok/missing    [OK / BLOCK]    (upload only)
```

---

## SUBCOMMAND: build

### Step 1 — Determine the git short SHA (for image naming)

```bash
cd "$CAA_ROOT"
GIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
echo "Git SHA: $GIT_SHA"
```

### Step 2 — Build binaries with TEE_PLATFORM=amd

```bash
cd "$CAA_ROOT/podvm-mkosi"
TEE_PLATFORM=amd PODVM_DISTRO=ubuntu make binaries
```

This builds kata-agent, attestation-agent (with AMD SNP attester), and other guest components into a Docker image and extracts them to `resources/binaries-tree/`. The `TEE_PLATFORM=amd` flag causes the attestation-agent to be built with AMD SNP attestation support instead of the default `none`.

Wait for completion — this typically takes 5–15 minutes depending on Docker layer cache state.

### Step 3 — Build debug mkosi image

```bash
cd "$CAA_ROOT/podvm-mkosi"
PODVM_DISTRO=ubuntu make image-debug
```

This builds the mkosi image with the `debug` profile (SSH support, verbose boot, debug tools) inside Docker using `--security=insecure`. It produces:
- `build/system.raw` — raw disk image (used for AMI upload)
- `build/podvm-ubuntu-amd64.qcow2` — qcow2 conversion (used for libvirt)

The raw image is what we need for the AMI upload path.

### Step 4 — Verify build output

```bash
ls -lh "$CAA_ROOT/podvm-mkosi/build/system.raw"
qemu-img info "$CAA_ROOT/podvm-mkosi/build/system.raw" | head -5
```

Print summary:
```
Build complete.
TEE_PLATFORM: amd
Profile:      debug
Raw image:    $CAA_ROOT/podvm-mkosi/build/system.raw
Size:         <N> GiB
Git SHA:      <sha>

Next step: /aws-podvm-ami upload --bucket <your-bucket> [--region <region>]
```

---

## SUBCOMMAND: upload

### Step 0 — Validate inputs

```bash
REGION="${REGION_ARG:-$AWS_REGION}"
[[ -n "$BUCKET_NAME" ]] || { echo "ERROR: --bucket is required for upload"; exit 1; }
[[ -n "$REGION" ]] || { echo "ERROR: --region or AWS_REGION must be set"; exit 1; }
```

Resolve the raw image path:
```bash
RAW_IMAGE="$CAA_ROOT/podvm-mkosi/build/system.raw"
[[ -f "$RAW_IMAGE" ]] || { echo "ERROR: $RAW_IMAGE not found. Run 'build' first."; exit 1; }
```

Resolve AMI name:
```bash
GIT_SHA=$(cd "$CAA_ROOT" && git rev-parse --short HEAD 2>/dev/null || echo "unknown")
AMI_NAME="${CUSTOM_NAME:-podvm-ubuntu-amd64-debug-${GIT_SHA}}"
echo "AMI name will be: $AMI_NAME"
```

### Step 1 — Set up TMPDIR with cleanup trap

```bash
TMPDIR=$(mktemp -d)
trap "rm -rf '$TMPDIR'" EXIT

IMAGE_IMPORT_JSON="$TMPDIR/image-import.json"
AMI_REGISTER_JSON="$TMPDIR/register-ami.json"
TRUST_POLICY_JSON="$TMPDIR/trust-policy.json"
ROLE_POLICY_JSON="$TMPDIR/role-policy.json"
BUCKET_POLICY_JSON="$TMPDIR/bucket-policy.json"
```

### Step 2 — Create S3 bucket (idempotent)

```bash
echo "Checking S3 bucket..."
if aws s3api head-bucket --bucket "$BUCKET_NAME" --region "$REGION" 2>/dev/null; then
    echo "Bucket $BUCKET_NAME already exists — skipping creation"
else
    echo "Creating S3 bucket $BUCKET_NAME in $REGION"
    if [[ "$REGION" == "us-east-1" ]]; then
        aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$REGION"
    else
        aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$REGION" \
            --create-bucket-configuration LocationConstraint="$REGION"
    fi
fi
```

### Step 3 — Create IAM role and policies (idempotent)

The `vmimport` role may already exist from a prior run with a *different* bucket. Always update the role's inline policy to include the current bucket, regardless of whether the role was just created or pre-existed.

```bash
echo "Checking vmimport IAM role..."
if aws iam get-role --role-name vmimport &>/dev/null; then
    echo "IAM role vmimport already exists — checking bucket scope ..."
    # Read existing S3 resource list and merge new bucket in if not already present
    EXISTING_RESOURCES=$(aws iam get-role-policy --role-name vmimport --policy-name vmimport \
        2>/dev/null | jq -r '.PolicyDocument.Statement[] | select(.Action[]? | contains("s3:GetObject")) | .Resource[]' || echo "")
    if echo "$EXISTING_RESOURCES" | grep -q "$BUCKET_NAME"; then
        echo "Bucket $BUCKET_NAME already in vmimport policy — no update needed"
    else
        echo "Adding $BUCKET_NAME to vmimport policy ..."
        # Build merged resource list: preserve existing + add new bucket
        EXISTING_JSON=$(aws iam get-role-policy --role-name vmimport --policy-name vmimport \
            2>/dev/null | jq '.PolicyDocument')
        MERGED=$(echo "$EXISTING_JSON" | jq \
            --arg b "arn:aws:s3:::${BUCKET_NAME}" \
            --arg bw "arn:aws:s3:::${BUCKET_NAME}/*" \
            '(.Statement[] | select(.Action[]? | contains("s3:GetObject")) | .Resource) += [$b, $bw]')
        echo "$MERGED" > "$ROLE_POLICY_JSON"
        aws iam put-role-policy --role-name vmimport --policy-name vmimport \
            --policy-document "file://$ROLE_POLICY_JSON"
        echo "vmimport policy updated."
    fi
else
    echo "Creating vmimport IAM role..."
    cat > "$TRUST_POLICY_JSON" <<EOF
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": { "Service": "vmie.amazonaws.com" },
        "Action": "sts:AssumeRole",
        "Condition": { "StringEquals": { "sts:Externalid": "vmimport" } }
    }]
}
EOF
    aws iam create-role --role-name vmimport \
        --assume-role-policy-document "file://$TRUST_POLICY_JSON" \
        --region "$REGION"

    cat > "$ROLE_POLICY_JSON" <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:GetBucketLocation","s3:GetObject","s3:ListBucket"],
            "Resource": ["arn:aws:s3:::${BUCKET_NAME}","arn:aws:s3:::${BUCKET_NAME}/*"]
        },
        {
            "Effect": "Allow",
            "Action": ["ec2:ModifySnapshotAttribute","ec2:CopySnapshot","ec2:RegisterImage","ec2:Describe*"],
            "Resource": "*"
        }
    ]
}
EOF
    aws iam put-role-policy --role-name vmimport --policy-name vmimport \
        --policy-document "file://$ROLE_POLICY_JSON"
fi

# Bucket policy is idempotent (put-bucket-policy is always safe to re-run)
cat > "$BUCKET_POLICY_JSON" <<EOF
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "AllowVMIE",
        "Effect": "Allow",
        "Principal": { "Service": "vmie.amazonaws.com" },
        "Action": ["s3:GetBucketLocation","s3:GetObject","s3:ListBucket"],
        "Resource": ["arn:aws:s3:::${BUCKET_NAME}","arn:aws:s3:::${BUCKET_NAME}/*"]
    }]
}
EOF
aws s3api put-bucket-policy --bucket "$BUCKET_NAME" \
    --policy "file://$BUCKET_POLICY_JSON" --region "$REGION"
```

### Step 4 — Upload raw image to S3

```bash
IMAGE_FILENAME="$(basename "$RAW_IMAGE")"
echo "Uploading $IMAGE_FILENAME to s3://$BUCKET_NAME/ ..."
aws s3 cp "$RAW_IMAGE" "s3://${BUCKET_NAME}/${IMAGE_FILENAME}" \
    --region "$REGION" \
    --no-progress \
    --expected-size "$(stat -c%s "$RAW_IMAGE")"
echo "Upload complete."
```

### Step 5 — Import snapshot and wait

```bash
echo "Importing snapshot from s3://$BUCKET_NAME/$IMAGE_FILENAME ..."

cat > "$IMAGE_IMPORT_JSON" <<EOF
{
    "Description": "Peer Pod VM image (debug, TEE_PLATFORM=amd)",
    "Format": "RAW",
    "UserBucket": {
        "S3Bucket": "${BUCKET_NAME}",
        "S3Key": "${IMAGE_FILENAME}"
    }
}
EOF

IMPORT_TASK_ID=$(aws ec2 import-snapshot \
    --disk-container "file://$IMAGE_IMPORT_JSON" \
    --output json --region "$REGION" | jq -r '.ImportTaskId')

[[ -n "$IMPORT_TASK_ID" ]] || { echo "ERROR: Failed to start import task"; exit 1; }
echo "Import task ID: $IMPORT_TASK_ID"

ITERATION=0
MAX_ITERATIONS=120  # 120 × 15s = 30 minutes
while [ $ITERATION -lt $MAX_ITERATIONS ]; do
    TASK_JSON=$(aws ec2 describe-import-snapshot-tasks \
        --import-task-ids "$IMPORT_TASK_ID" \
        --output json --region "$REGION")
    IMPORT_STATUS=$(echo "$TASK_JSON" | jq -r '.ImportSnapshotTasks[0].SnapshotTaskDetail.Status')
    IMPORT_MSG=$(echo "$TASK_JSON" | jq -r '.ImportSnapshotTasks[0].SnapshotTaskDetail.StatusMessage // ""')

    echo "[$ITERATION] Import status: $IMPORT_STATUS${IMPORT_MSG:+ / $IMPORT_MSG}"

    if [[ "$IMPORT_STATUS" == "completed" ]]; then
        echo "Snapshot import completed."
        break
    elif [[ "$IMPORT_STATUS" != "active" && "$IMPORT_STATUS" != "pending" ]]; then
        echo "ERROR: Import failed with status '$IMPORT_STATUS'"; exit 2
    fi

    ITERATION=$(( ITERATION + 1 ))
    sleep 15
done

if [ $ITERATION -eq $MAX_ITERATIONS ]; then
    echo "ERROR: Snapshot import timed out after 30 minutes"; exit 1
fi

SNAPSHOT_ID=$(aws ec2 describe-import-snapshot-tasks \
    --import-task-ids "$IMPORT_TASK_ID" \
    --output json --region "$REGION" | \
    jq -r '.ImportSnapshotTasks[0].SnapshotTaskDetail.SnapshotId')

[[ -n "$SNAPSHOT_ID" && "$SNAPSHOT_ID" != "null" ]] || \
    { echo "ERROR: Could not retrieve snapshot ID from completed import task"; exit 1; }

echo "Snapshot ID: $SNAPSHOT_ID"
aws ec2 wait snapshot-completed --snapshot-ids "$SNAPSHOT_ID" --region "$REGION"
echo "Snapshot ready."
```

### Step 6 — Register AMI

```bash
echo "Registering AMI from snapshot $SNAPSHOT_ID ..."

BUILD_DATE=$(date -u +%Y-%m-%d)

cat > "$AMI_REGISTER_JSON" <<EOF
{
    "Architecture": "x86_64",
    "BlockDeviceMappings": [{
        "DeviceName": "/dev/xvda",
        "Ebs": {
            "DeleteOnTermination": true,
            "SnapshotId": "${SNAPSHOT_ID}"
        }
    }],
    "Description": "Peer-pod debug image (TEE_PLATFORM=amd, Ubuntu 24.04)",
    "RootDeviceName": "/dev/xvda",
    "VirtualizationType": "hvm",
    "EnaSupport": true,
    "BootMode": "uefi"
}
EOF

AMI_ID=$(aws ec2 register-image \
    --name "$AMI_NAME" \
    --cli-input-json "file://$AMI_REGISTER_JSON" \
    --tpm-support v2.0 \
    --output json --region "$REGION" | jq -r '.ImageId')

[[ -n "$AMI_ID" && "$AMI_ID" != "null" ]] || \
    { echo "ERROR: AMI registration failed"; exit 1; }

echo "AMI registered: $AMI_ID"
```

### Step 7 — Tag the AMI

Collect the required tagging inputs before running:
- `AMI_NAME` — the `Name` tag value (e.g. `fedora-mkosi-debug-tee-amd`)
- `KEEP_BECAUSE` — `"$userid - $reason"` (e.g. `"bpradipt - upstream coco podvm image"`)
- `CAA_VERSION` — the CAA release version (e.g. `0.21.0`); read from the most recent `v*` git tag or upcoming release noted in commit messages

If not supplied as arguments, ask the user for `KEEP_BECAUSE` and `CAA_VERSION` before tagging.

```bash
echo "Tagging AMI $AMI_ID ..."
aws ec2 create-tags --resources "$AMI_ID" --region "$REGION" --tags \
    Key=Name,Value="$AMI_NAME" \
    Key=TEE_PLATFORM,Value=amd \
    Key=Profile,Value=debug \
    Key=Distro,Value=ubuntu-24.04 \
    Key=GitCommit,Value="$GIT_SHA" \
    Key=BuildDate,Value="$BUILD_DATE" \
    Key=keep-because,Value="$KEEP_BECAUSE" \
    Key=Version,Value="$CAA_VERSION"

echo "Tags applied."
```

### Step 8 — Optional S3 cleanup

If `--cleanup` was passed:
```bash
echo "Deleting $IMAGE_FILENAME from s3://$BUCKET_NAME/ ..."
aws s3 rm "s3://${BUCKET_NAME}/${IMAGE_FILENAME}" --region "$REGION"
echo "S3 object deleted."
```

If not `--cleanup`, remind the user:
```
Note: The raw image remains in s3://$BUCKET_NAME/$IMAGE_FILENAME
      Delete it manually when no longer needed to avoid storage costs:
      aws s3 rm s3://$BUCKET_NAME/$IMAGE_FILENAME --region $REGION
```

### Step 9 — Print upload summary

```
Upload complete.
  AMI name:     $AMI_NAME
  AMI ID:       $AMI_ID
  Region:       $REGION
  Snapshot:     $SNAPSHOT_ID
  TEE_PLATFORM: amd
  Profile:      debug
  Git SHA:      $GIT_SHA
  Build date:   $BUILD_DATE
```

---

## SUBCOMMAND: all

Run `build` then `upload` in sequence. Collect `--bucket`, `--region`, `--name`, `--cleanup` args and pass them to the upload phase.

After build completes successfully, proceed to upload without re-prompting unless the upload args are missing.

</process>

<success_criteria>
**build:** `build/system.raw` exists with size > 1 GB and `qemu-img info` confirms it is a valid raw image
**upload:** `aws ec2 describe-images --image-ids <AMI_ID>` returns the registered image with state `available`
</success_criteria>
