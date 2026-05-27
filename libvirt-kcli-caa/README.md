# libvirt-kcli-caa command

Sets up and tests [cloud-api-adaptor](https://github.com/confidential-containers/cloud-api-adaptor) with libvirt locally. Creates the kcli peer-pods Kubernetes cluster, uploads the podvm qcow2, installs CAA via Helm, and runs e2e tests.

Use `/libvirt-podvm-qcow2` to build the podvm image before running `setup`.

## Invocation

```
/libvirt-kcli-caa [setup [--image <path>] | test [--filter <TestRegex>] [--debug] [--kbs] | teardown]
```

Run with no arguments for an interactive prompt.

## Subcommands

| Subcommand | What it does |
|---|---|
| `setup` | Installs system dependencies (libvirt, kcli, kubectl, helm, Go), creates the peer-pods K8s cluster, uploads the podvm qcow2 to the libvirt pool, and installs CAA via Helm |
| `test` | Runs libvirt e2e tests against an existing cluster with CAA installed |
| `teardown` | Uninstalls CAA, deletes the cluster, and cleans up libvirt volumes |

## Flags

| Flag | Applies to | Effect |
|---|---|---|
| `--image <path>` | `setup` | Path to the podvm qcow2 built by `/libvirt-podvm-qcow2`. Defaults to `$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2` if it exists. |
| `--debug` | `test` | Adds `-v` verbose output to the test run |
| `--filter <regex>` | `test` | Go regex passed to `go test -run`. Use alternation for multiple tests: `(TestLibvirtFoo\|TestLibvirtBar)`. |
| `--kbs` | `test` | Sets `DEPLOY_KBS=yes`, deploying the Key Broker Service and enabling KBS-related tests (e.g. `TestLibvirtKbsKeyRelease`). Requires `oras` and `kustomize`. |

## Expected repo layout

```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/
        ├── Makefile
        ├── libvirt/
        │   ├── config_libvirt.sh
        │   ├── kcli_cluster.sh
        │   └── install_caa.sh
        ├── libvirt.properties
        ├── podvm-mkosi/build/podvm-ubuntu-amd64.qcow2   ← default image path
        └── test/e2e/
```

**KUBECONFIG:** `~/.kcli/clusters/peer-pods/auth/kubeconfig`

## Known pitfalls

### Expected test failure

`TestLibvirtCreatePeerPodWithLargeImage` always fails locally because `ENABLE_SCRATCH_SPACE=false`. This is intentional — it calls `SkipTestOnCI(t)` and is skipped in CI. A healthy run produces **23 PASS, 1 FAIL**.

### KBS cert mismatch after failed or repeated runs

The test setup generates a fresh TLS cert pair on every run. If a previous run left stale state, the certs can mismatch and KBS tests will fail. Run `teardown` to clean up before retrying.

## Install

```bash
cp libvirt-kcli-caa/libvirt-kcli-caa.md ~/.claude/commands/libvirt-kcli-caa.md
```
