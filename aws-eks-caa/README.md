# aws-eks-caa command

Manages the full lifecycle of a Cloud API Adaptor (CAA) peer-pods deployment on AWS EKS: cluster creation, CAA installation via Helm, test workload verification, and teardown.

## Invocation

```
/aws-eks-caa [setup [--ami <AMI_ID>] [--region <REGION>] [--cluster-name <NAME>] [--instance-type <TYPE>] [--non-cvm] | test | teardown [--cluster-name <NAME>]]
```

Run with no arguments for an interactive prompt.

## Subcommands

| Subcommand | What it does |
|---|---|
| `setup` | Creates an EKS cluster, configures security group rules, installs fuse on worker nodes, and deploys CAA via the peerpods Helm chart |
| `test` | Deploys an nginx pod with `runtimeClassName: kata-remote` and verifies the corresponding EC2 peer-pod VM appears |
| `teardown` | Deletes the nginx workload, uninstalls the Helm chart, and deletes the EKS cluster |

## Flags

| Flag | Applies to | Effect |
|---|---|---|
| `--ami <id>` | `setup` | **Required.** AMI ID of the podvm image to use |
| `--region <region>` | `setup`, `teardown` | AWS region (default: `us-east-2`) |
| `--cluster-name <name>` | `setup`, `teardown` | EKS cluster name (default: `caa-<timestamp>`) |
| `--instance-type <type>` | `setup` | EC2 instance type for podvms (default: `m6a.large`) |
| `--non-cvm` | `setup` | Disables CVM mode (`DISABLECVM=true`); switches podvm instance to `t3.large` for non-confidential testing |

## Prerequisites

- AWS credentials in the environment (`AWS_PROFILE`, instance role, or env vars)
- `aws`, `kubectl`, `helm`, `jq` installed
- `eksctl` — installed automatically from GitHub releases if missing
- The CAA Helm chart at `~/cloud-api-adaptor/src/cloud-api-adaptor/install/charts/peerpods/`

## Expected repo layout

```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/
        └── install/
            └── charts/
                └── peerpods/
                    ├── Chart.yaml
                    ├── values.yaml
                    └── providers/
                        └── aws.yaml        ← written by setup
```

## Defaults

| Parameter | Default |
|---|---|
| Region | `us-east-2` |
| EKS node type | `m5.xlarge` |
| PodVM instance type | `m6a.large` (AMD SEV-SNP) |
| CAA version | `0.21.0` |
| CVM mode | enabled |

## Known pitfalls

### eksctl must be installed from GitHub releases
`apt` packages are often stale. The preflight step installs the latest release automatically.

### Stale CloudFormation stack from interrupted run
If `eksctl create cluster` is interrupted it leaves a CF stack that blocks re-runs with the same name. Fix: use a new cluster name (timestamp suffix) or delete the stale stack manually:
```bash
aws cloudformation update-termination-protection --no-enable-termination-protection \
    --stack-name "eksctl-<old-name>-cluster" --region $REGION
aws cloudformation delete-stack --stack-name "eksctl-<old-name>-cluster" --region $REGION
aws cloudformation wait stack-delete-complete --stack-name "eksctl-<old-name>-cluster" --region $REGION
```

### AL2023 worker nodes missing `mount.fuse`
`kata-remote` uses the `nydus-for-kata-tee` snapshotter which requires `mount.fuse` on the host. AL2023 doesn't include it. The setup step deploys a privileged DaemonSet to install `fuse` via `dnf` before any peer-pod workloads are scheduled.

### Clock skew breaks EC2 API calls
SigV4 requires the system clock to be within 5 minutes of AWS time. The preflight step checks and corrects skew automatically using the `aws.amazon.com` Date header.

### Security group rules already exist
`authorize-security-group-ingress` errors if a rule already exists. These errors are suppressed — the setup step treats them as non-fatal.

### CAA image tag must match a published release
Verify before use: `curl -sI "https://quay.io/v2/confidential-containers/cloud-api-adaptor/manifests/v0.21.0-amd64" -o /dev/null -w "%{http_code}"`. Use `v0.20.0-amd64` if the target version returns `404`.

### AWS_SESSION_TOKEN required for temporary credentials
If the profile uses STS assumed-role credentials, `AWS_SESSION_TOKEN` must be included in the Kubernetes secret or CAA will get `ExpiredToken` errors. The setup step reads it from the profile automatically.

## Install

```bash
cp aws-eks-caa/aws-eks-caa.md ~/.claude/commands/aws-eks-caa.md
```
