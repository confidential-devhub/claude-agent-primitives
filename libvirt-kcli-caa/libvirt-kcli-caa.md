---
name: libvirt-kcli-caa
description: Set up and test cloud-api-adaptor with libvirt locally. Creates the kcli peer-pods cluster, uploads the podvm qcow2, installs CAA, and runs e2e tests. Use /libvirt-podvm-qcow2 to build the podvm image first.
argument-hint: "setup [--image <path>] | test [--filter <TestRegex>] [--debug] [--kbs] | teardown"
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

- **setup** — install system dependencies, create the kcli peer-pods cluster, upload the podvm qcow2 to the libvirt pool, and install CAA via Helm
- **test** — run libvirt e2e tests against an already-running cluster with CAA installed
- **teardown** — uninstall CAA, delete the cluster, and clean up libvirt volumes

`--image <path>` on `setup`: path to a podvm qcow2 built by `/libvirt-podvm-qcow2`. Defaults to `$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2` if it exists.
`--filter <regex>` on `test`: Go regex passed to `go test -run`. Use alternation for multiple: `(TestLibvirtFoo|TestLibvirtBar)`.
`--debug` on `test`: adds `-v` verbose output.
`--kbs` on `test`: sets `DEPLOY_KBS=yes`, deploying the Key Broker Service and enabling KBS-related tests (e.g. `TestLibvirtKbsKeyRelease`). Requires `oras` and `kustomize`.

Architecture is fixed at **amd64**. Distro is fixed at **Ubuntu 24.04**.
</objective>

<repo_layout>
```
~/cloud-api-adaptor/
└── src/
    └── cloud-api-adaptor/          ← CAA_ROOT
        ├── Makefile
        ├── libvirt/
        │   ├── config_libvirt.sh   ← installs deps + tools
        │   ├── kcli_cluster.sh     ← creates/deletes K8s cluster via kcli
        │   └── install_caa.sh      ← installs CAA via Helm into cluster
        ├── libvirt.properties      ← test provisioner config (TOML)
        ├── install/charts/peerpods/providers/libvirt.yaml  ← Helm values (modified by install_caa.sh)
        ├── podvm-mkosi/build/podvm-ubuntu-amd64.qcow2      ← default image path
        └── test/e2e/               ← e2e test suite
```

**KUBECONFIG:** `~/.kcli/clusters/peer-pods/auth/kubeconfig`
**Cluster nodes:** peer-pods-ctlplane-0 (control-plane), peer-pods-worker-0 (worker)
**Cluster network:** 192.168.123.0/24 (actual bridge IP varies — always detect, never assume)
</repo_layout>

<known_pitfalls>

## 1. Expected test failure — not a real failure
`TestLibvirtCreatePeerPodWithLargeImage` fails locally because `ENABLE_SCRATCH_SPACE=false`
in the libvirt config. It calls `SkipTestOnCI(t)` and is intentionally skipped in CI.
Do not treat this as a regression. Expected result: **23 PASS, 1 FAIL** on a healthy setup.

## 2. KBS cert mismatch after failed or repeated runs

The test setup generates a fresh TLS cert pair on every run and writes it to
`test/trustee/kbs/config/kubernetes/base/https-cert.pem`. If an old KBS pod from a prior
run is still running (because the `coco-tenant` namespace didn't finish terminating),
`kbs-client` fails with `SSL: certificate verify failed (self-signed certificate)`.

**Fix:** Force-delete the namespace and confirm it is gone before re-running:

```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
kubectl delete namespace coco-tenant --ignore-not-found --force --grace-period=0
until ! kubectl get namespace coco-tenant 2>/dev/null; do sleep 2; done
echo "coco-tenant namespace gone"
```

Also note: `generateCert()` is called **twice** per test run, each time overwriting
`https-cert.pem`. The cert deployed to the KBS pod is from the first call; the file on
disk after the run is from the second. This is a framework bug — the workaround is simply
ensuring no stale KBS pod is running before starting.

</known_pitfalls>

<process>

## 0. Parse Arguments

Parse `$ARGUMENTS`:
- First positional arg: subcommand (`setup`, `test`, `teardown`)
- `--image <path>`: path to podvm qcow2 (setup only; default: `$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2`)
- `--filter <regex>`: test name regex (test only)
- `--debug`: verbose test output (test only)
- `--kbs`: deploy Key Broker Service and run KBS tests (test only)

If no subcommand given, ask:
```
Which phase do you want to run?
  1) setup    — create cluster, upload podvm image, install CAA
  2) test     — run e2e tests (setup must already be complete)
  3) teardown — uninstall CAA and delete the cluster
```

## 1. Locate the Repo

```bash
ls ~/cloud-api-adaptor/src/cloud-api-adaptor/Makefile 2>/dev/null
```

If not found, use AskUserQuestion to ask for the path. Set `CAA_ROOT` to the resolved path.

---

## 2. Preflight Resource Check (all subcommands)

```bash
DISK_REPO=$(df -BG --output=avail "$CAA_ROOT" | tail -1 | tr -d 'G ')
DISK_LIBVIRT=$(df -BG --output=avail /var/lib/libvirt 2>/dev/null | tail -1 | tr -d 'G ' || echo 0)
RAM_TOTAL=$(free -g | awk '/^Mem:/ {print $2}')
RAM_AVAIL=$(free -g | awk '/^Mem:/ {print $7}')
CPU_CORES=$(nproc)
KVM_OK=$(test -e /dev/kvm && echo "present" || echo "missing")
```

| Resource | Minimum | Action if below |
|---|---|---|
| Disk (repo/Docker) | 20 GB | BLOCK |
| Disk (/var/lib/libvirt) | 10 GB | BLOCK |
| RAM total | 8 GB | WARN |
| RAM available | 4 GB | WARN |
| KVM (/dev/kvm) | required | BLOCK for setup/test |

Print summary and stop on any BLOCK.

---

## SUBCOMMAND: setup

### Step 1 — Resolve podvm image path

```bash
PODVM_IMAGE="${IMAGE_ARG:-$CAA_ROOT/podvm-mkosi/build/podvm-ubuntu-amd64.qcow2}"
[[ -f "$PODVM_IMAGE" ]] || {
    echo "ERROR: podvm image not found at $PODVM_IMAGE"
    echo "Run /libvirt-podvm-qcow2 first to build the image."
    exit 1
}
echo "Using podvm image: $PODVM_IMAGE"
```

### Step 2 — System dependencies

```bash
sudo apt-get update -qq
sudo apt-get install -y \
    bubblewrap dnf qemu-utils uidmap alien \
    libvirt-daemon-system libvirt-clients virtinst cpu-checker
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt "$USER"
```

Check tools (install if missing):

```bash
# Go
go version 2>/dev/null || echo "WARNING: Go not found — needed for test subcommand"

# kcli
which kcli 2>/dev/null || curl -s https://raw.githubusercontent.com/karmab/kcli/main/install.sh | sudo sh

# kubectl
which kubectl 2>/dev/null || (
    curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl
)

# helm
which helm 2>/dev/null || curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Step 3 — Create kcli cluster

Check if cluster already exists first:
```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
READY=$(kubectl get nodes --no-headers 2>/dev/null | grep -c Ready || echo 0)
```

If `READY < 2`:
```bash
cd "$CAA_ROOT"
bash libvirt/config_libvirt.sh
bash libvirt/kcli_cluster.sh create
```

Verify:
```bash
kubectl get nodes
```
Expected: peer-pods-ctlplane-0 (control-plane) and peer-pods-worker-0 (worker) both Ready.

### Step 4 — Upload podvm image to libvirt pool

Upload the qcow2 as both `podvm-base.qcow2` (primary volume used by CAA) and
`another-podvm-base.qcow2` (used by multi-pod tests). This is a local libvirt instance,
so use `virsh -c qemu:///system` directly — no SSH needed.

```bash
for VOL in podvm-base.qcow2 another-podvm-base.qcow2; do
    virsh -c qemu:///system vol-delete "$VOL" default 2>/dev/null || true
    virsh -c qemu:///system vol-create-as default "$VOL" 20G \
        --format qcow2 --allocation 2G
    virsh -c qemu:///system vol-upload --pool default "$VOL" "$PODVM_IMAGE"
    echo "Uploaded $VOL"
done
virsh -c qemu:///system vol-list default | grep podvm
```

### Step 5 — Install CAA via install_caa.sh

`install_caa.sh` modifies `install/charts/peerpods/providers/libvirt.yaml` in-place to set
`LIBVIRT_URI`, then runs `make CLOUD_PROVIDER=libvirt deploy` (Helm install).

Detect the libvirt bridge IP — do NOT assume 192.168.122.1, always detect:
```bash
LIBVIRT_IP=$(virsh -c qemu:///system net-dumpxml default | sed -n "s/.*ip address='\(.*\)' .*/\1/p")
echo "Libvirt bridge IP: $LIBVIRT_IP"
```

Determine SSH user (root may be disabled; try ubuntu first):
```bash
SSH_KEY=~/.ssh/id_rsa
if ssh -i "$SSH_KEY" -o BatchMode=yes -o ConnectTimeout=5 ubuntu@"$LIBVIRT_IP" "true" 2>/dev/null; then
    LIBVIRT_USER=ubuntu
elif ssh -i "$SSH_KEY" -o BatchMode=yes -o ConnectTimeout=5 root@"$LIBVIRT_IP" "true" 2>/dev/null; then
    LIBVIRT_USER=root
else
    echo "ERROR: cannot SSH to $LIBVIRT_IP as ubuntu or root"
    exit 1
fi
echo "SSH user: $LIBVIRT_USER"
```

Run the installer:
```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
export LIBVIRT_IP
export LIBVIRT_USER
export SSH_KEY_FILE="$SSH_KEY"
cd "$CAA_ROOT"
bash libvirt/install_caa.sh
```

### Step 6 — Persist state

```bash
echo "peer-pods"            > /tmp/caa_libvirt_cluster_name
echo "$LIBVIRT_IP"          > /tmp/caa_libvirt_ip
echo "$LIBVIRT_USER"        > /tmp/caa_libvirt_user
echo "$PODVM_IMAGE"         > /tmp/caa_libvirt_image
```

### Step 7 — Verify

```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
echo "=== Nodes ==="
kubectl get nodes
echo ""
echo "=== CAA pods ==="
kubectl get pods -n confidential-containers-system
echo ""
echo "=== RuntimeClass ==="
kubectl get runtimeclass
```

Expected: `kata-remote` runtimeClass present, CAA daemonset pod Running on worker node.

Print summary:
```
Setup complete.
  Cluster:    peer-pods
  KUBECONFIG: ~/.kcli/clusters/peer-pods/auth/kubeconfig
  PodVM image uploaded: podvm-base.qcow2, another-podvm-base.qcow2
  CAA:        installed and running

Next steps:
  /libvirt-kcli-caa test          — run e2e tests
  /libvirt-kcli-caa teardown      — clean up when done
```

---

## SUBCOMMAND: test

### Step 1 — Restore context and verify prerequisites

```bash
CLUSTER_NAME=$(cat /tmp/caa_libvirt_cluster_name 2>/dev/null || echo "peer-pods")
LIBVIRT_IP=$(cat /tmp/caa_libvirt_ip 2>/dev/null || \
    virsh -c qemu:///system net-dumpxml default | sed -n "s/.*ip address='\(.*\)' .*/\1/p")
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig

READY=$(kubectl get nodes --no-headers 2>/dev/null | grep -c Ready || echo 0)
[ "$READY" -ge 2 ] || { echo "ERROR: Cluster not ready ($READY nodes). Run setup first."; exit 1; }

kubectl get pods -n confidential-containers-system --no-headers 2>/dev/null | grep -q Running \
    || echo "WARN: CAA pods not found — was setup run? Continuing anyway."
```

### Step 2 — KBS prerequisites (only if --kbs)

**Install oras and kustomize if missing:**
```bash
# oras — version pinned in versions.yaml
ORAS_VERSION=$(grep -A1 'name: oras' "$CAA_ROOT/../../versions.yaml" 2>/dev/null | grep version | awk '{print $2}' \
    || echo "1.2.0")
command -v oras &>/dev/null || (
    curl -sLO "https://github.com/oras-project/oras/releases/download/v${ORAS_VERSION}/oras_${ORAS_VERSION}_linux_amd64.tar.gz"
    tar -xzf "oras_${ORAS_VERSION}_linux_amd64.tar.gz"
    sudo install -m 0755 oras /usr/local/bin/oras
    rm -f oras oras_*.tar.gz
)

command -v kustomize &>/dev/null || (
    curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
    sudo install -m 0755 kustomize /usr/local/bin/kustomize
    rm -f kustomize
)
```

**Run checkout_kbs.sh** (one-time per repo clone; re-run if versions.yaml changes):
```bash
cd "$CAA_ROOT"
bash test/utils/checkout_kbs.sh
```

**Force-terminate any stale KBS namespace** (see pitfall #2):
```bash
kubectl delete namespace coco-tenant --ignore-not-found --force --grace-period=0 2>/dev/null || true
until ! kubectl get namespace coco-tenant &>/dev/null; do sleep 2; done
echo "coco-tenant namespace gone"
```

### Step 3 — Verify or create libvirt.properties

**If the file already exists:** read it and show `libvirt_uri` — ask whether to keep or update. Never silently overwrite.

**If missing:** create it. The file is TOML — all string values must be quoted.

Discover bridge IP:
```bash
LIBVIRT_IP=$(cat /tmp/caa_libvirt_ip 2>/dev/null || \
    virsh -c qemu:///system net-dumpxml default | sed -n "s/.*ip address='\(.*\)' .*/\1/p")
```

Determine SSH user (check /tmp state first, then probe):
```bash
LIBVIRT_USER=$(cat /tmp/caa_libvirt_user 2>/dev/null || echo "ubuntu")
```

Write:
```toml
libvirt_uri = "qemu+ssh://<user>@<bridge_ip>/system?no_verify=1"
libvirt_ssh_key_file = "/home/ubuntu/.ssh/id_rsa"
cluster_name = "peer-pods"
```

Only include fields that differ from provisioner defaults. Do not add `container_runtime`,
`pause_image`, `tunnel_type`, `vxlan_port`, `libvirt_network`, `libvirt_storage`, or
`libvirt_conn_uri` unless explicitly needed.

### Step 4 — Clean up leftover namespaces from prior runs

```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
kubectl get namespaces | awk '/coco-pp-e2e/{print $1}' | \
    xargs -r kubectl delete namespace --ignore-not-found 2>/dev/null || true
```

### Step 5 — Run e2e tests

CAA is already installed by setup — set `TEST_INSTALL_CAA=no` and `TEST_PROVISION=no`.

```bash
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
export CLOUD_PROVIDER=libvirt
export TEST_PROVISION=no
export TEST_INSTALL_CAA=no
export TEST_TEARDOWN=no
export TEST_E2E_TIMEOUT=75m
export TEST_PROVISION_FILE="$CAA_ROOT/libvirt.properties"
```

If `--kbs`:
```bash
export DEPLOY_KBS=yes
```

If `--filter <regex>`:
```bash
export RUN_TESTS="<regex>"
```

**Run using `go test` directly** — not `make test-e2e`, because the Makefile passes
`RUN_TESTS` unquoted so `|` is interpreted as a shell pipe:

```bash
cd "$CAA_ROOT"
go test -v -tags=libvirt \
    -timeout "${TEST_E2E_TIMEOUT:-75m}" \
    -count=1 \
    -run "${RUN_TESTS:-.}" \
    ./test/e2e
```

If `--debug`, add `-v` (already included above).

### Step 6 — Parse and report results

- Total tests run, passed, failed
- List failures by test name
- If `TestLibvirtCreatePeerPodWithLargeImage` is the only failure: note this is expected (pitfall #1)
- If other tests fail: show test name and last relevant error lines

---

## SUBCOMMAND: teardown

### Step 1 — Restore context

```bash
CLUSTER_NAME=$(cat /tmp/caa_libvirt_cluster_name 2>/dev/null || echo "peer-pods")
export KUBECONFIG=~/.kcli/clusters/peer-pods/auth/kubeconfig
echo "Will tear down cluster: $CLUSTER_NAME"
```

### Step 2 — Uninstall CAA

```bash
helm uninstall peerpods -n confidential-containers-system 2>/dev/null || true
kubectl delete namespace confidential-containers-system --ignore-not-found --wait --timeout=60s
kubectl get namespaces | awk '/coco-pp-e2e|coco-tenant/{print $1}' | \
    xargs -r kubectl delete namespace --ignore-not-found --force --grace-period=0 2>/dev/null || true
echo "CAA uninstalled."
```

### Step 3 — Delete libvirt volumes

```bash
for VOL in podvm-base.qcow2 another-podvm-base.qcow2; do
    virsh -c qemu:///system vol-delete "$VOL" default 2>/dev/null \
        && echo "Deleted volume: $VOL" \
        || echo "Volume $VOL not found (already deleted)"
done
```

### Step 4 — Delete kcli cluster

```bash
cd "$CAA_ROOT"
CLUSTER_NAME="$CLUSTER_NAME" bash libvirt/kcli_cluster.sh delete
echo "Cluster deleted."
```

### Step 5 — Clean up temp state

```bash
rm -f /tmp/caa_libvirt_cluster_name /tmp/caa_libvirt_ip \
      /tmp/caa_libvirt_user /tmp/caa_libvirt_image
echo "Teardown complete."
```

</process>

<success_criteria>
**setup:** `kubectl get nodes` shows 2 Ready nodes; `kubectl get runtimeclass` shows `kata-remote`; CAA daemonset pod Running in `confidential-containers-system`
**test:** e2e suite completes; failures limited to `TestLibvirtCreatePeerPodWithLargeImage` only (expected)
**teardown:** cluster deleted; no podvm volumes remain in libvirt default pool
</success_criteria>
