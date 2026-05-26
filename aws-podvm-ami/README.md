# aws-podvm-ami command

Builds a debug Ubuntu podvm mkosi image with `TEE_PLATFORM=amd` (AMD SEV-SNP attestation) and registers it as an AWS AMI. Can be run as separate `build` and `upload` phases or as a single `all` sequence.

## Invocation

```
/aws-podvm-ami [build | upload --bucket <S3_BUCKET> [--region <REGION>] [--name <AMI_NAME>] [--cleanup] | all --bucket <S3_BUCKET> [--region <REGION>] [--name <AMI_NAME>] [--cleanup]]
```

Run with no arguments for an interactive prompt.

## Subcommands

| Subcommand | What it does |
|---|---|
| `build` | Compiles guest binaries with `TEE_PLATFORM=amd`, then builds a debug mkosi image; produces `build/system.raw` |
| `upload` | Uploads an existing `build/system.raw` to S3, imports it as an EBS snapshot, and registers it as an AMI |
| `all` | Runs `build` then `upload` in sequence |

## Flags

| Flag | Applies to | Effect |
|---|---|---|
| `--bucket <name>` | `upload`, `all` | **Required.** S3 bucket name for the raw image upload |
| `--region <region>` | `upload`, `all` | AWS region (falls back to `$AWS_REGION`) |
| `--name <ami-name>` | `upload`, `all` | Custom AMI name (default: `podvm-ubuntu-amd64-debug-<git-sha>`) |
| `--cleanup` | `upload`, `all` | Deletes the raw image from S3 after successful AMI registration |

## Prerequisites

- Docker running with sufficient disk space (~20 GB free)
- AWS credentials in the environment (`AWS_PROFILE`, instance role, or env vars)
- `aws`, `jq` installed
- The CAA repo at `~/cloud-api-adaptor/src/cloud-api-adaptor/`

## Expected repo layout

```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/
        ├── Makefile.defaults
        └── podvm-mkosi/
            ├── Makefile
            └── build/
                ├── system.raw                  ← raw image (AMI input)
                └── podvm-ubuntu-amd64.qcow2    ← qcow2 (libvirt only)
```

## Build output

| Artifact | Location | Used for |
|---|---|---|
| `system.raw` | `podvm-mkosi/build/system.raw` | AMI upload |
| `podvm-ubuntu-amd64.qcow2` | `podvm-mkosi/build/podvm-ubuntu-amd64.qcow2` | libvirt |

## AMI tags applied

| Tag | Value |
|---|---|
| `Name` | AMI name |
| `TEE_PLATFORM` | `amd` |
| `Profile` | `debug` |
| `Distro` | `ubuntu-24.04` |
| `GitCommit` | Short SHA of HEAD |
| `BuildDate` | UTC date |
| `keep-because` | `<userid> - <reason>` (prompted if not supplied) |
| `Version` | CAA release version (prompted if not supplied) |

## Known pitfalls

### Clock skew breaks EC2 API calls
SigV4 requires the system clock within 5 minutes of AWS time. The upload preflight checks and corrects skew automatically using the `aws.amazon.com` Date header.

### vmimport IAM role scope
The `vmimport` role may exist from a prior run but its inline policy may only cover a different S3 bucket. The upload step merges the new bucket into the existing policy rather than failing or recreating the role.

### S3 bucket in us-east-1
Bucket creation in `us-east-1` must omit the `LocationConstraint` parameter — other regions require it. The upload step handles this automatically.

### Snapshot import timeout
The import poll loop runs for up to 30 minutes (120 × 15 s). Large raw images (~1.5 GB) typically complete in 5–10 minutes.

## Install

```bash
cp aws-podvm-ami/aws-podvm-ami.md ~/.claude/commands/aws-podvm-ami.md
```
