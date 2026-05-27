# libvirt-podvm-qcow2 command

Builds the Ubuntu 24.04 podvm disk image for libvirt (amd64). Compiles kata-agent, agent-protocol-forwarder, and other guest binaries first, then builds the qcow2 via mkosi inside Docker.

## Invocation

```
/libvirt-podvm-qcow2 [--debug]
```

Run with no arguments for a production build.

## Flags

| Flag | Effect |
|---|---|
| `--debug` | Uses `make image-debug` — includes SSH/SFTP access, verbose systemd boot, and additional diagnostic utilities. Use for troubleshooting podvm issues. |

## Output

`$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2`

This image is consumed by `/libvirt-kcli-caa setup`.

## Expected repo layout

```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/
        └── podvm-mkosi/
            ├── Makefile           ← binaries / image / image-debug targets
            └── build/
                ├── system.raw                   ← intermediate raw image
                └── podvm-ubuntu-amd64.qcow2     ← final output
```

## Resource requirements

| Resource | Minimum | Notes |
|---|---|---|
| Free disk | 20 GB | Docker layers (~5 GB) + system.raw (~3 GB) + qcow2 (~1 GB) |
| Available RAM | 4 GB | mkosi Docker build can spike |

Docker must be running before invoking this command.

## Build stages

1. **Binaries** — `PODVM_DISTRO=ubuntu make binaries` compiles kata-agent, agent-protocol-forwarder, and other guest components via a multi-stage Docker build. Takes ~5–10 minutes on first run; subsequent runs use the Docker layer cache.
2. **Image** — `PODVM_DISTRO=ubuntu make image` (or `make image-debug`) runs mkosi inside Docker with `--security=insecure`. Produces `build/system.raw` and automatically converts it to `build/podvm-ubuntu-amd64.qcow2`. Takes ~5–15 minutes depending on package download speed.

## Install

```bash
cp libvirt-podvm-qcow2/libvirt-podvm-qcow2.md ~/.claude/commands/libvirt-podvm-qcow2.md
```
