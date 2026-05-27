---
name: libvirt-kcli-caa
description: Set up, build, and test cloud-api-adaptor with libvirt locally. Builds Ubuntu podvm via mkosi (amd64), manages the libvirt/kcli cluster, and runs e2e tests with optional test filtering and debug image support.
argument-hint: "setup | build [--debug] | test [--filter <TestRegex>] [--debug] [--kbs]"
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
Manage the full local dev/test lifecycle for cloud-api-adaptor on libvirt (amd64):

- **setup** — install system dependencies, verify libvirt, create the kcli peer-pods cluster
- **build** — compile kata-agent + agent-protocol-forwarder binaries, then build Ubuntu podvm via mkosi, convert to qcow2
- **test** — run libvirt e2e tests; optionally target a specific test by name regex

`--debug` on `build` uses `make image-debug` (includes debugging tools, SSH, verbose boot) instead of production image.
`--debug` on `test` adds `-v` verbose test output.
`--kbs` on `test` sets `DEPLOY_KBS=yes`, which deploys the Key Broker Service and enables KBS-related tests (e.g. `TestLibvirtKbsKeyRelease`). Requires `oras` and `kustomize` to be installed.

Architecture is fixed at **amd64**. Distro is fixed at **Ubuntu 24.04**.
</objective>

<repo_layout>
The canonical repo layout expected on this machine:

```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/          ← CAA_ROOT
        ├── Makefile
        ├── libvirt/
        │   ├── config_libvirt.sh   ← installs deps + tools
        │   ├── kcli_cluster.sh     ← creates K8s cluster via kcli
        │   └── install_caa.sh      ← installs CAA into cluster
        ├── libvirt.properties      ← test provisioner config
        ├── podvm-mkosi/            ← mkosi build workspace
        │   ├── Makefile            ← make image / make image-debug
        │   ├── mkosi.presets/system/mkosi.conf.d/ubuntu.conf  ← package list
        │   ├── mkosi.postinst      ← post-install hook (resolv.conf symlink)
        │   └── build/
        │       ├── system.raw      ← raw output from mkosi
        │       └── podvm-ubuntu-amd64.qcow2  ← final qcow2
        └── test/e2e/               ← e2e test suite
```

**KUBECONFIG:** `~/.kcli/clusters/peer-pods/auth/kubeconfig`
**Cluster nodes:** peer-pods-ctlplane-0, peer-pods-worker-0
**Cluster network:** 192.168.123.0/24
</repo_layout>

<known_pitfalls>
CRITICAL — encode these checks before any build:

## 1. Missing packages in ubuntu.conf
File: `podvm-mkosi/mkosi.presets/system/mkosi.conf.d/ubuntu.conf`

Both of these MUST be in the `Packages:` list:
- `ca-certificates` — without it, CDH (Rust binary using reqwest/native-tls) cannot make TLS connections to container registries; image pulls silently fail
- `systemd-resolved` — Ubuntu ships this as a separate package from `systemd`; without it DNS resolution fails in the podvm (nsswitch.conf reads `/etc/resolv.conf` → stub resolver)

## 2. Missing resolv.conf symlink
File: `podvm-mkosi/mkosi.postinst`

This block must be present:
```bash
if [ ! -e "${BUILDROOT}/etc/resolv.conf" ]; then
    ln -s /run/systemd/resolve/stub-resolv.conf "${BUILDROOT}/etc/resolv.conf" || true
fi
```

## 3. kata-agent / CDH startup race — `image_guest_pull` always fails with KBS initdata

When a pod uses initdata (e.g. KBS tests with `cc_kbc`), the startup sequence is:
`process-user-data` → AA socket → CDH starts → CDH socket.
`kata-agent` also starts after `process-user-data` and probes the CDH socket immediately.
Because CDH isn't ready yet, kata-agent permanently marks CDH as `None` and panics
on every `image_guest_pull` with **"Confidential Data Hub not initialized"**.

**Fix:** Add `confidential-data-hub.service` to kata-agent's `After=` and `Wants=` so it
waits for CDH to fully start before probing.

File: `podvm-mkosi/resources/binaries-tree/etc/systemd/system/kata-agent.service`

```ini
[Unit]
After=netns@podns.service process-user-data.service scratch-storage.service confidential-data-hub.service
Wants=scratch-storage.service confidential-data-hub.service
```

After editing this file, **rebuild the podvm image** before running tests.

## 4. Expected test failure — not a real failure
`TestLibvirtCreatePeerPodWithLargeImage` fails locally because `ENABLE_SCRATCH_SPACE=false`
in the libvirt config. It calls `SkipTestOnCI(t)` and is intentionally skipped in CI.
Do not treat this as a regression. Expected result: **23 PASS, 1 FAIL** on a healthy setup.

## 5. Storage volumes missing when TEST_PROVISION=no

`CreateVPC()` — which creates the `podvm-base.qcow2` and `another-podvm-base.qcow2` volumes
in the libvirt default pool — is only called when `TEST_PROVISION=yes`. When the cluster
already exists (`TEST_PROVISION=no`), it is skipped, and `UploadPodvm()` will fail with:

```
virError(Code=50): Storage volume not found: no storage vol with matching name 'podvm-base.qcow2'
```

**Fix:** Before running tests with `TEST_PROVISION=no`, check that the volumes exist and
create them if missing:

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<libvirt_host> "
  virsh -c qemu:///system vol-list default | grep podvm-base.qcow2 ||
  virsh -c qemu:///system vol-create-as default podvm-base.qcow2 20G --format qcow2 --allocation 2G
  virsh -c qemu:///system vol-list default | grep another-podvm-base.qcow2 ||
  virsh -c qemu:///system vol-create-as default another-podvm-base.qcow2 20G --format qcow2 --allocation 2G
"
```

The `<libvirt_host>` is the IP in `libvirt_uri` from `libvirt.properties`.

## 6. KBS cert mismatch after failed or repeated runs

The test setup generates a fresh TLS cert pair on every run and writes it to
`test/trustee/kbs/config/kubernetes/base/https-cert.pem`. It also deploys a new KBS pod
using that cert. But **if an old KBS pod from a prior run is still running** (because the
`coco-tenant` namespace didn't finish terminating), the new cert is written to disk while
the pod continues serving the old cert. `kbs-client` then fails with:

```
SSL: certificate verify failed (self-signed certificate)
```

**Fix:** Force-delete the namespace and confirm it is gone before re-running:

```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
kubectl delete namespace coco-tenant --ignore-not-found --force --grace-period=0
# Wait until fully gone
until ! kubectl get namespace coco-tenant 2>/dev/null; do sleep 2; done
echo "coco-tenant namespace gone"
```

Also note: `generateCert()` is called **twice** per test run (once in `NewKeyBrokerService()`
and once inside `Deploy()`), each time overwriting `https-cert.pem`. The cert that ends up
in `https-cert.pem` is from the second call, while the cert actually deployed to the KBS pod
is from the first. This is a bug in the test framework — the workaround is simply ensuring
no stale KBS pod is running before the test starts.
</known_pitfalls>

<process>

## 0. Parse Arguments

Parse `$ARGUMENTS`:
- First positional arg: subcommand (`setup`, `build`, `test`)
- `--debug`: boolean flag, applies to build (use `make image-debug`) and test (add `-v`)
- `--filter <regex>`: test name regex passed to `go test -run` (test subcommand only). Use Go regex alternation to run multiple tests: `(TestLibvirtFoo|TestLibvirtBar)` — the parentheses are optional but make intent clear.
- `--kbs`: boolean flag (test subcommand only) — sets `DEPLOY_KBS=yes` to deploy the Key Broker Service and enable KBS-related tests

If no subcommand given, ask:
```
Which phase do you want to run?
  1) setup  — install deps and create libvirt/kcli cluster
  2) build  — build binaries and podvm qcow2
  3) test   — run e2e tests (cluster + image must already exist)
```

## 1. Locate the Repo

Check default path first:
```bash
ls ~/cloud-api-adaptor/src/cloud-api-adaptor/Makefile 2>/dev/null
```

If not found, use AskUserQuestion:
> "Where is the cloud-api-adaptor repo? (Expected: ~/cloud-api-adaptor — provide the full path to the directory containing the top-level Makefile)"

Set `CAA_ROOT` to the resolved path. All subsequent commands use `CAA_ROOT`.

---

## 2. Preflight Resource Check (all subcommands)

Run before executing any subcommand. Gather the numbers:

```bash
# Free disk on the partition containing CAA_ROOT (and /var/lib/docker, /var/lib/libvirt)
df -BG --output=avail "$CAA_ROOT" | tail -1 | tr -d 'G '   # GB free on repo partition
df -BG --output=avail /var/lib/docker 2>/dev/null | tail -1 | tr -d 'G '   # GB free for Docker
df -BG --output=avail /var/lib/libvirt 2>/dev/null | tail -1 | tr -d 'G '  # GB free for libvirt

# Total and available RAM
free -g | awk '/^Mem:/ {print $2, $7}'   # total_GB available_GB

# CPU cores
nproc

# KVM acceleration
ls /dev/kvm 2>/dev/null && echo "kvm_ok" || echo "kvm_missing"
```

Evaluate and report:

| Resource | Minimum | Recommended | Action if below minimum |
|---|---|---|---|
| Free disk (repo/Docker partition) | 20 GB | 40 GB | BLOCK — builds and Docker layer cache alone exceed this |
| Free disk (/var/lib/libvirt) | 10 GB | 20 GB | BLOCK — each VM disk + podvm image volume live here |
| Total RAM | 8 GB | 16 GB | WARN — 2 libvirt VMs + host + peer-pod spawns will be tight |
| Available RAM | 4 GB | 8 GB | WARN — risk of OOM during mkosi Docker build or VM startup |
| CPU cores | 2 | 4+ | INFO only — build will be slow but not blocked |
| KVM (/dev/kvm) | required | required | BLOCK for `setup`/`test`; WARN for `build` only |

**Disk note:** The breakdown of space consumed:
- Docker build layers for binaries image: ~4–6 GB
- mkosi `system.raw` intermediate file: ~2–4 GB
- Final `podvm-ubuntu-amd64.qcow2`: ~1–2 GB
- libvirt base volume (`podvm-base.qcow2`) uploaded during test: ~2 GB
- Per-test peer-pod VM disk (cloned from base): ~2 GB × number of concurrent tests
- Total for a single full run: **~15–20 GB minimum**

**KVM note:** Without `/dev/kvm`, libvirt falls back to software emulation — peer-pod VMs will be unusably slow. This is a functional block for `setup` and `test`. For `build` only (no VM spawned), warn but do not block.

Print a preflight summary before proceeding:
```
Preflight check:
  Disk (repo):    <N> GB free   [OK / WARN / BLOCK]
  Disk (libvirt): <N> GB free   [OK / WARN / BLOCK]
  RAM total:      <N> GB        [OK / WARN]
  RAM available:  <N> GB        [OK / WARN]
  CPU cores:      <N>           [OK / INFO]
  KVM:            present/missing [OK / BLOCK]
```

If any BLOCK condition is hit, stop and tell the user what needs to be resolved. Do not proceed.
If only WARN conditions, print them clearly and ask the user to confirm before continuing.

---

## SUBCOMMAND: setup

### Step 1 — System dependencies

```bash
sudo apt-get update -qq
sudo apt-get install -y \
    bubblewrap \
    dnf \
    qemu-utils \
    uidmap \
    alien \
    libvirt-daemon-system \
    libvirt-clients \
    virtinst \
    cpu-checker
```

Check libvirt daemon is running:
```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER
```

### Step 2 — Go (if not present)

```bash
go version 2>/dev/null || (
  echo "Go not found — check versions.yaml for required version"
  grep -i "^go:" "$CAA_ROOT/../../versions.yaml" 2>/dev/null || echo "Install Go manually: https://go.dev/dl/"
)
```

### Step 3 — kcli (if not present)

```bash
which kcli 2>/dev/null || curl -s https://raw.githubusercontent.com/karmab/kcli/main/install.sh | sudo sh
```

### Step 4 — kubectl (if not present)

```bash
which kubectl 2>/dev/null || (
  curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
  rm kubectl
)
```

### Step 5 — Helm (if not present)

```bash
which helm 2>/dev/null || curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Step 6 — Create the kcli cluster

Run the setup scripts from the libvirt directory:
```bash
cd "$CAA_ROOT"
bash libvirt/config_libvirt.sh
bash libvirt/kcli_cluster.sh
```

### Step 7 — Verify cluster

```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
kubectl get nodes
```

Expected: peer-pods-ctlplane-0 (control-plane) and peer-pods-worker-0 (worker) both Ready.

Print:
```
Setup complete.
KUBECONFIG: ~/.kcli/clusters/peer-pods/auth/kubeconfig

Next step: /libvirt-kcli-caa build
```

---

## SUBCOMMAND: build

### Step 1 — Check known pitfalls BEFORE building

Read `$CAA_ROOT/podvm-mkosi/mkosi.presets/system/mkosi.conf.d/ubuntu.conf` and verify:
- `ca-certificates` appears in the `Packages:` block
- `systemd-resolved` appears in the `Packages:` block

Read `$CAA_ROOT/podvm-mkosi/mkosi.postinst` and verify the `resolv.conf` symlink block is present.

**If any check fails:** Show what is missing and offer to fix it automatically before proceeding. These are silent runtime failures — the image will build but the podvm will malfunction.

### Step 2 — Build binaries

```bash
cd "$CAA_ROOT/podvm-mkosi"
PODVM_DISTRO=ubuntu make binaries
```

The `binaries` target lives in `podvm-mkosi/Makefile`, not the top-level Makefile. It builds kata-agent, agent-protocol-forwarder, and other guest components into a Docker image and extracts them to `resources/binaries-tree/`. Wait for completion — this can take several minutes.

### Step 3 — Build mkosi image

```bash
cd "$CAA_ROOT/podvm-mkosi"
```

If `--debug`:
```bash
PODVM_DISTRO=ubuntu make image-debug
```

If not `--debug` (production):
```bash
PODVM_DISTRO=ubuntu make image
```

The debug profile includes: debugging tools, SSH/SFTP support, verbose systemd boot, additional diagnostic utilities.

This step runs mkosi inside Docker with `--security=insecure`. It produces `build/system.raw` **and automatically converts it to `build/podvm-ubuntu-amd64.qcow2`** — no separate conversion step needed.

### Step 4 — Verify output

```bash
ls -lh "$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2"
qemu-img info "$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2"
```

Print build summary:
```
Build complete.
Image: $CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2
Mode: [production|debug]

Next step: /libvirt-kcli-caa test
```

---

## SUBCOMMAND: test

### Step 1 — Verify prerequisites

Check that the cluster is up:
```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
kubectl get nodes --no-headers 2>/dev/null | grep -c Ready
```

If 0 nodes ready, warn: "Cluster not running. Run `/libvirt-kcli-caa setup` first."

Check image exists:
```bash
ls "$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2" 2>/dev/null
```

If missing, warn: "podvm image not found. Run `/libvirt-kcli-caa build` first."

**KBS tests require infrastructure setup before `make test-e2e`** — this mirrors the CI "Checkout KBS Repository" step and must be done manually for local runs.

#### KBS infrastructure setup (one-time per repo clone)

**1. Install prerequisites:**
```bash
# oras — use version pinned in versions.yaml
ORAS_VERSION=$(yq -e '.tools.oras' "$CAA_ROOT/../../versions.yaml")
curl -sLO "https://github.com/oras-project/oras/releases/download/v${ORAS_VERSION}/oras_${ORAS_VERSION}_linux_amd64.tar.gz"
tar -xzf oras_${ORAS_VERSION}_linux_amd64.tar.gz && sudo install -m 0755 oras /usr/local/bin/oras

# kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo install -m 0755 kustomize /usr/local/bin/kustomize
```

**2. Run `checkout_kbs.sh`** (same script that CI runs as a dedicated step):
```bash
cd "$CAA_ROOT"
bash test/utils/checkout_kbs.sh
```

This script:
- Clones the trustee repo at the commit pinned in `versions.yaml` into `test/trustee/`
- Installs `kbs-client` binary via `oras pull` from GHCR → `test/trustee/kbs-client`
- Patches `kbs-config.toml`: replaces `insecure_http=true` with HTTPS cert/key path settings
- Adds Deployment volume-mount patches and cert secretGenerators to `base/kustomization.yaml`

The test provisioner's `Deploy()` then generates the actual HTTPS cert (`generateCert()`), writes it into the placeholder files the script created, and `Apply()` bakes everything into the KBS manifests via kustomize. No further manual setup is needed between `checkout_kbs.sh` and `make test-e2e`.

Re-run `checkout_kbs.sh` whenever `versions.yaml` changes the trustee pinned commit.

### Step 2 — Verify or create libvirt.properties

**File format:** The provisioner parses this file with `toml.Unmarshal` — string values must be TOML-quoted. All fields below are optional except `libvirt_uri` and `libvirt_ssh_key_file`; the rest have working defaults in the provisioner code (`libvirt_network=default`, `libvirt_storage=default`, `libvirt_vol_name=podvm-base.qcow2`, `libvirt_conn_uri=qemu:///system`, `cluster_name=peer-pods`).

**If `libvirt.properties` already exists:** Read it and show the current `libvirt_uri` to the user — ask whether to keep it or update it. Never silently overwrite an existing file.

**If missing or needs correction, follow these steps:**

#### A. Discover the libvirt bridge IP

```bash
ip -4 addr show | grep -A1 virbr
```

The bridge IP (e.g. `192.168.123.1`) is the address VMs use to reach the host over the NAT network. Do NOT assume `192.168.122.1` — the actual IP depends on which libvirt network was created and varies per machine.

#### B. Determine the SSH user for qemu+ssh

The `libvirt_uri` uses `qemu+ssh://` which SSHes into the host. `root` SSH access is often disabled. Test both:

```bash
ssh -i ~/.ssh/id_rsa -o BatchMode=yes -o ConnectTimeout=5 root@<bridge_ip> "virsh list" 2>&1
ssh -i ~/.ssh/id_rsa -o BatchMode=yes -o ConnectTimeout=5 ubuntu@<bridge_ip> "virsh list" 2>&1
```

Use whichever user succeeds (typically `ubuntu` when root SSH is disabled).

#### C. Write the properties file

```toml
libvirt_uri = "qemu+ssh://<user>@<bridge_ip>/system?no_verify=1"
libvirt_ssh_key_file = "/home/ubuntu/.ssh/id_rsa"
cluster_name = "peer-pods"
```

Only include fields that differ from defaults. Do not add `container_runtime`, `pause_image`, `tunnel_type`, `vxlan_port`, `libvirt_network`, `libvirt_storage`, or `libvirt_conn_uri` unless explicitly overriding them — the provisioner defaults are correct for a standard local setup.

### Step 3 — Construct and run the test command

**`TEST_PROVISION` controls cluster creation, not image upload or CAA install.** Always check whether a cluster already exists before setting it:

```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
READY=$(kubectl get nodes --no-headers 2>/dev/null | grep -c Ready || true)
if [ "$READY" -ge 2 ]; then
  export TEST_PROVISION=no   # cluster already exists — skip kcli_cluster.sh create
else
  export TEST_PROVISION=yes  # provision a new cluster
fi
```

Image upload (`TEST_PODVM_IMAGE`) and CAA installation (`TEST_INSTALL_CAA`) are **independent** of `TEST_PROVISION` — they run regardless and are always needed on a fresh run.

Base environment:
```bash
export CLOUD_PROVIDER=libvirt
export TEST_TEARDOWN=no
export TEST_INSTALL_CAA=yes
export TEST_E2E_TIMEOUT=75m
export TEST_PODVM_IMAGE="$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2"
export TEST_PROVISION_FILE="$CAA_ROOT/libvirt.properties"
```

If KBS tests are requested (`--kbs` or `DEPLOY_KBS=yes`):
```bash
export DEPLOY_KBS=yes
```

If `--filter <regex>` was provided:
```bash
export RUN_TESTS="<regex>"
```

If `--debug`:
```bash
export TEST_VERBOSE=true
```

Before running, clean up any leftover CAA installation from a previous run (avoids "namespace already exists" errors):
```bash
helm uninstall peerpods -n confidential-containers-system 2>/dev/null || true
kubectl delete namespace confidential-containers-system --ignore-not-found --wait=true --timeout=60s
kubectl get namespaces | awk '/coco-pp-e2e/{print $1}' | xargs -r kubectl delete namespace --ignore-not-found 2>/dev/null || true
```

If `DEPLOY_KBS=yes`, also fully terminate the KBS namespace before starting (see pitfall #6):
```bash
kubectl delete namespace coco-tenant --ignore-not-found --force --grace-period=0 2>/dev/null || true
until ! kubectl get namespace coco-tenant &>/dev/null; do sleep 2; done
echo "coco-tenant namespace gone"
```

Pre-create libvirt volumes if they don't exist (required when `TEST_PROVISION=no`, see pitfall #5).
Parse `libvirt_uri` from `libvirt.properties` to get the host, then:
```bash
LIBVIRT_HOST=$(grep libvirt_uri "$CAA_ROOT/libvirt.properties" | grep -oP '(?<=@)[^/]+')
SSH_KEY=$(grep libvirt_ssh_key_file "$CAA_ROOT/libvirt.properties" | grep -oP '"[^"]+"' | tr -d '"')
for VOL in podvm-base.qcow2 another-podvm-base.qcow2; do
  ssh -i "$SSH_KEY" -o BatchMode=yes ubuntu@"$LIBVIRT_HOST" \
    "virsh -c qemu:///system vol-list default | grep -q $VOL || \
     virsh -c qemu:///system vol-create-as default $VOL 20G --format qcow2 --allocation 2G" 2>&1
done
```

**Run using `go test` directly** (not `make test-e2e`). The Makefile passes `RUN_TESTS`
unquoted, so a `|` in the filter value is interpreted as a shell pipe. `go test` with a
single-quoted `-run` argument avoids this:

```bash
cd "$CAA_ROOT"
go test -v -tags=libvirt \
  -timeout "${TEST_E2E_TIMEOUT:-75m}" \
  -count=1 \
  -run "${RUN_TESTS:-.}" \
  ./test/e2e
```

When `RUN_TESTS` is unset or empty this runs all tests (`.` matches everything).
When set (e.g. `RUN_TESTS='(TestLibvirtKbsKeyRelease|TestLibvirtSealedSecret)'`), the
shell single-quoting in the export means the `|` reaches `go test` as a regex alternation,
not a pipe.

### Step 4 — Parse and report results

After the test run completes, parse the output for PASS/FAIL counts.

Report:
- Total tests run, passed, failed
- List any failures by test name
- If `TestLibvirtCreatePeerPodWithLargeImage` is the only failure: note this is expected (requires `ENABLE_SCRATCH_SPACE=true`, intentionally skipped in CI)
- If other tests fail: show the test name and last relevant error lines

</process>

<success_criteria>
**setup:** `kubectl get nodes` shows 2 Ready nodes (ctlplane + worker)
**build:** `build/podvm-ubuntu-amd64.qcow2` exists and `qemu-img info` reports a valid qcow2
**test:** e2e suite completes; failures limited to known expected failures only
</success_criteria>
