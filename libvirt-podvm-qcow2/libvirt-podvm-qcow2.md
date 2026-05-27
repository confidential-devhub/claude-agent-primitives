---
name: libvirt-podvm-qcow2
description: Build a Ubuntu podvm qcow2 image for libvirt (amd64) using mkosi. Compiles kata-agent and agent-protocol-forwarder binaries first, then builds the disk image. Output is podvm-mkosi/build/podvm-ubuntu-amd64.qcow2.
argument-hint: "[--debug]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
---

<objective>
Build the Ubuntu 24.04 podvm disk image for libvirt (amd64):

1. Compile kata-agent, agent-protocol-forwarder, and other guest binaries into a Docker image and extract them
2. Build the disk image with mkosi inside Docker, producing a qcow2

`--debug`: builds with the debug profile — includes SSH/SFTP access, verbose systemd boot, and additional diagnostic utilities. Use for troubleshooting podvm issues.

Output: `$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2`

This image is consumed by `/libvirt-kcli-caa setup --image <path>`.
</objective>

<repo_layout>
```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/          ← CAA_ROOT
        └── podvm-mkosi/            ← all build targets run here
            ├── Makefile
            │   ├── binaries        ← builds guest binaries via Docker
            │   ├── image           ← builds production qcow2 via mkosi
            │   └── image-debug     ← builds debug qcow2 via mkosi
            ├── mkosi.images/
            │   └── system/
            │       ├── mkosi.conf.d/ubuntu.conf   ← package list
            │       ├── mkosi.postinst             ← post-install hook
            │       └── mkosi.finalize.chroot      ← resolv.conf symlink
            └── build/
                ├── system.raw                     ← intermediate raw image
                └── podvm-ubuntu-amd64.qcow2       ← final output
```
</repo_layout>

<process>

## 0. Parse Arguments

- `--debug`: use `make image-debug` instead of `make image`

## 1. Locate the Repo

```bash
ls ~/cloud-api-adaptor/src/cloud-api-adaptor/Makefile 2>/dev/null
```

Set `CAA_ROOT` to the resolved path.

---

## 2. Preflight Resource Check

```bash
DISK_REPO=$(df -BG --output=avail "$CAA_ROOT" | tail -1 | tr -d 'G ')
RAM_AVAIL=$(free -g | awk '/^Mem:/ {print $7}')
CPU_CORES=$(nproc)
```

| Resource | Minimum | Action |
|---|---|---|
| Free disk | 20 GB | BLOCK — Docker layers (~5 GB) + system.raw (~3 GB) + qcow2 (~1 GB) |
| Available RAM | 4 GB | WARN — mkosi Docker build can spike |

Docker must be running:
```bash
docker info &>/dev/null || { echo "BLOCK: Docker not running"; exit 1; }
```

Print summary and stop on any BLOCK.

---

## 3. Build Binaries

```bash
cd "$CAA_ROOT/podvm-mkosi"
echo "Building guest binaries (PODVM_DISTRO=ubuntu) ..."
PODVM_DISTRO=ubuntu make binaries
```

This runs a multi-stage Docker build that compiles kata-agent, agent-protocol-forwarder,
and other guest components, then extracts them to `resources/binaries-tree/`.
Takes ~5–10 minutes on first run (cached on subsequent runs).

---

## 4. Build mkosi Image

```bash
cd "$CAA_ROOT/podvm-mkosi"
```

If `--debug`:
```bash
echo "Building debug podvm image ..."
PODVM_DISTRO=ubuntu make image-debug
```

Otherwise (production):
```bash
echo "Building production podvm image ..."
PODVM_DISTRO=ubuntu make image
```

This runs mkosi inside Docker with `--security=insecure`. It produces `build/system.raw`
and automatically converts it to `build/podvm-ubuntu-amd64.qcow2`. No separate conversion
step is needed.

Takes ~5–15 minutes depending on package download speed.

---

## 5. Verify Output

```bash
ls -lh "$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2"
qemu-img info "$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2"
```

Print summary:
```
Build complete.
  Image: $CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2
  Mode:  [production | debug]
  Size:  <reported by qemu-img>

Next step:
  /libvirt-kcli-caa setup                          — use default image path
  /libvirt-kcli-caa setup --image <path>           — use explicit path
```

</process>

<success_criteria>
`$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2` exists and `qemu-img info` reports a valid qcow2 image.
</success_criteria>
