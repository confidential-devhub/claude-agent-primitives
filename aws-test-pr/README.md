# aws-test-pr command

Tests a cloud-api-adaptor GitHub PR against an existing AWS EKS peer-pods cluster. Inspects the PR diff to determine what needs rebuilding (CAA container image, podvm AMI, or both), builds and pushes those artifacts, deploys them to the running cluster, and runs a smoke test to confirm a peer-pod EC2 VM is created. Supports an incremental `--rebuild-podvm` mode for iterating on the podvm image without a full rerun.

## Invocation

```
/aws-test-pr <PR_NUMBER> [--caa] [--podvm] [--rebuild-podvm] [--no-build] [--test-filter <regex>] [--cleanup]
```

## Flags

| Flag | Effect |
|---|---|
| `--caa` | Force rebuild of the CAA container image even if the diff doesn't require it |
| `--podvm` | Force rebuild of the podvm image/AMI even if the diff doesn't require it |
| `--rebuild-podvm` | Iteration mode: deregister the existing PR AMI, rebuild from the checked-out branch, register a new AMI, update `peer-pods-cm`, clean-restart CAA, re-run the smoke test. Does not touch the CAA binary image. Requires a prior full run. |
| `--no-build` | Skip all builds; re-deploy and test with whatever images already exist |
| `--test-filter <regex>` | Run `go test -run <regex>` against `./test/e2e` instead of the default nginx smoke test |
| `--cleanup` | After testing, automatically run cleanup (revert CAA image, restore original AMI, delete test pods and PR artifacts) |

## Rebuild detection

The command fetches the PR's changed-file list from the GitHub API and sets rebuild flags automatically:

| Changed path prefix | Triggers |
|---|---|
| `src/cloud-api-adaptor/` or `src/cloud-providers/` | CAA image rebuild |
| `src/cloud-api-adaptor/podvm-mkosi/` | PodVM image rebuild |
| `src/cloud-api-adaptor/pkg/forwarder/`, `cmd/agent-protocol-forwarder/`, `cmd/process-user-data/`, `podvm/Dockerfile.podvm_binaries` | PodVM image rebuild (guest binaries) |

`--caa` and `--podvm` flags override detection; `--no-build` suppresses both.

## Process overview

| Step | Action |
|---|---|
| 0 | Parse arguments |
| 1 | Resolve environment (AWS account, ECR registry, kubeconfig, current cluster state) |
| 2 | Inspect PR via GitHub API — fetch metadata and changed-file list, determine rebuild plan |
| 3 | Check out the PR branch (`git fetch origin pull/<N>/head:pr-<N>`) |
| 4 | Build CAA container image and push to ECR (if needed) |
| 5 | Build debug podvm image, upload to S3, register as AMI (if needed) |
| 6 | Deploy — patch daemonset image and/or `peer-pods-cm` configmap |
| 7 | Test — nginx peer-pod smoke test or `go test` e2e run |
| 8 | `--rebuild-podvm` iteration loop — deregister old AMI, rebuild, re-register, update cluster, re-test |
| 10 | Cleanup — restore original CAA image, original AMI, delete PR ECR tag and AMI, restore git branch |

## Prerequisites

- AWS credentials in the environment (`AWS_PROFILE`, instance role, or env vars)
- `aws`, `kubectl`, `docker`, `jq`, `curl` installed
- EKS cluster already running with CAA installed — set up via `/aws-eks-caa setup`
- State files from that setup: `/tmp/caa_eks_region`, `/tmp/caa_eks_kubeconfig`
- The CAA repo at `~/cloud-api-adaptor/src/cloud-api-adaptor/`

## Expected repo layout

```
~/cloud-api-adaptor/
└── src/
    ├── cloud-api-adaptor/     ← CAA binary, podvm-mkosi, e2e tests
    └── cloud-providers/       ← separate Go module compiled into CAA image
```

## State files written

| File | Contents |
|---|---|
| `/tmp/caa_pr_number` | PR number |
| `/tmp/caa_pr_original_image` | Pre-PR CAA daemonset image (for cleanup) |
| `/tmp/caa_pr_original_ami` | Pre-PR PODVM_AMI_ID from `peer-pods-cm` (for cleanup) |
| `/tmp/caa_pr_original_branch` | Git branch before checkout (for cleanup) |
| `/tmp/caa_pr_new_image` | ECR image tag built for this PR |
| `/tmp/caa_pr_new_ami` | AMI ID registered for this PR |

## Cleanup

Cleanup reverts all changes made during the test:

- Restores the original CAA daemonset image
- Restores the original `PODVM_AMI_ID` in `peer-pods-cm`
- Deregisters the PR AMI and deletes its EBS snapshot
- Deletes the PR ECR image tag
- Deletes the `nginx-pr-test` deployment
- Restores the original git branch and deletes the `pr-<N>` local branch
- Removes all `/tmp/caa_pr_*` state files

## Known pitfalls

### Wrong container name in `kubectl set image`
The CAA daemonset container is named `cloud-api-adaptor-con`, not `cloud-api-adaptor`. Always resolve the name dynamically before calling `kubectl set image`:
```bash
CONTAINER_NAME=$(kubectl get daemonset cloud-api-adaptor-daemonset \
    -n confidential-containers-system \
    -o jsonpath='{.spec.template.spec.containers[0].name}')
kubectl set image daemonset/cloud-api-adaptor-daemonset \
    "${CONTAINER_NAME}=${PR_CAA_IMAGE}" -n confidential-containers-system
```

### `kata.peerpods.io/vm` extended resource cleared by AL2023 kubelet
The AL2023 managed kubelet periodically overwrites the node status, removing the extended resource. Patch the node immediately before deploying any test pod:
```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl patch node "$NODE" --subresource=status --type=json \
    -p '[{"op":"add","path":"/status/capacity/kata.peerpods.io~1vm","value":"10"}]'
```

### CAA socket race during rolling restart
`/run/peerpod/hypervisor.sock` is a shared hostPath. During `rollout restart` the old pod's shutdown removes the socket after the new pod creates it. Use a clean stop/start instead:
```bash
kubectl patch daemonset cloud-api-adaptor-daemonset -n confidential-containers-system \
    --type json -p '[{"op":"replace","path":"/spec/template/spec/nodeSelector","value":{"cloud-api-adaptor-disabled":"true"}}]'
kubectl wait --for=delete pod -n confidential-containers-system -l app=cloud-api-adaptor --timeout=60s
kubectl patch daemonset cloud-api-adaptor-daemonset -n confidential-containers-system \
    --type json -p '[{"op":"replace","path":"/spec/template/spec/nodeSelector","value":{"node.kubernetes.io/worker":""}}]'
```

### GitHub API rate limit
Unauthenticated requests are limited to 60/hour per IP. Set `GH_TOKEN` and add `-H "Authorization: Bearer $GH_TOKEN"` to `curl` calls if hitting limits.

### PR branch from a fork
The `pull/<N>/head` ref works via the upstream `origin` remote regardless of whether the PR is from a fork — no need to add the fork as a separate remote.

### `go.sum` mismatch after branch switch
If `go.mod` changed on the PR branch, run `go mod tidy` in both `src/cloud-api-adaptor` and `src/cloud-providers` before building.

## Install

```bash
cp aws-test-pr/aws-test-pr.md ~/.claude/commands/aws-test-pr.md
```
