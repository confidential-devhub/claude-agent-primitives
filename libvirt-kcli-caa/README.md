# libvirt-kcli-caa command

Manages the full local dev/test lifecycle for [cloud-api-adaptor](https://github.com/confidential-containers/cloud-api-adaptor) on libvirt (amd64). Builds an Ubuntu 24.04 podvm image via mkosi, manages the libvirt/kcli Kubernetes cluster, and runs e2e tests.

## Invocation

```
/libvirt-kcli-caa [setup | build [--debug] | test [--filter <TestRegex>] [--debug]]
```

Run with no arguments for an interactive prompt.

## Subcommands

| Subcommand | What it does |
|---|---|
| `setup` | Installs system dependencies (libvirt, kcli, kubectl, helm, Go), creates the peer-pods K8s cluster |
| `build` | Compiles kata-agent + agent-protocol-forwarder, builds Ubuntu podvm via mkosi, converts to qcow2 |
| `test` | Runs libvirt e2e tests against an existing cluster and podvm image |

## Flags

| Flag | Applies to | Effect |
|---|---|---|
| `--debug` | `build` | Uses `make image-debug` — includes SSH, debugging tools, verbose boot |
| `--debug` | `test` | Adds `-v` verbose output to the test run |
| `--filter <regex>` | `test` | Sets `RUN_TESTS=<regex>` to target specific tests by name |

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
        ├── podvm-mkosi/
        │   ├── Makefile
        │   └── build/
        │       ├── system.raw
        │       └── podvm-ubuntu-amd64.qcow2
        └── test/e2e/
```

**KUBECONFIG:** `~/.kcli/clusters/peer-pods/auth/kubeconfig`

## Known pitfalls

### Missing packages in ubuntu.conf

`podvm-mkosi/mkosi.presets/system/mkosi.conf.d/ubuntu.conf` must include both:
- `ca-certificates` — required for TLS connections in CDH; image pulls silently fail without it
- `systemd-resolved` — required for DNS in the podvm; not bundled with `systemd` on Ubuntu

### Missing resolv.conf symlink

`podvm-mkosi/mkosi.postinst` must contain:
```bash
if [ ! -e "${BUILDROOT}/etc/resolv.conf" ]; then
    ln -s /run/systemd/resolve/stub-resolv.conf "${BUILDROOT}/etc/resolv.conf" || true
fi
```

The command checks for both of these automatically before every build and offers to fix them.

### Expected test failure

`TestLibvirtCreatePeerPodWithLargeImage` always fails locally because `ENABLE_SCRATCH_SPACE=false`. This is intentional — it calls `SkipTestOnCI(t)` and is skipped in CI. A healthy run produces **23 PASS, 1 FAIL**.

## Install

```bash
cp libvirt-kcli-caa/libvirt-kcli-caa.md ~/.claude/commands/libvirt-kcli-caa.md
```
