I have a ubuntu control plane node and running inside vmware as guest. I setup kubernetes and containerd and cilium but not working. many issue are there. so I want to remove all the things and start from starch. need to remove all the setup and this node should be fresh to start with new configurations.
## Summary

Complete teardown of Kubernetes, containerd, and Cilium from an Ubuntu node — resetting it to a clean pre-install state. This covers kubeadm reset, CNI cleanup, containerd purge, kernel module cleanup, network interface teardown, iptables/nftables flush, and residual config/data removal. Since this is a VMware guest control-plane node, we also handle VMware-specific networking artifacts. After this, the node will be ready for a clean reinstall.

---

## Phase 1 — Kubernetes Reset

```bash
# If kubeadm is still present, do a full reset first
sudo kubeadm reset --force --cri-socket unix:///run/containerd/containerd.sock

# Remove kubeadm, kubelet, kubectl
sudo apt-mark unhold kubeadm kubelet kubectl 2>/dev/null || true
sudo apt-get remove --purge -y kubeadm kubelet kubectl kubernetes-cni
sudo apt-get autoremove -y
```

---

## Phase 2 — Cilium & CNI Cleanup

```bash
# Remove cilium CLI if installed
sudo rm -f /usr/local/bin/cilium

# Tear down cilium network interfaces
for iface in $(ip link show | grep -oP '(?<=\d: )(cilium\S+|lxc\S+|cilium_\S+)'); do
  sudo ip link set "$iface" down 2>/dev/null
  sudo ip link delete "$iface" 2>/dev/null
done

# Remove CNI configs and binaries
sudo rm -rf /etc/cni/
sudo rm -rf /opt/cni/
sudo rm -rf /var/lib/cni/

# Remove CNI binaries if present separately
sudo rm -f /usr/lib/cni/*
```

---

## Phase 3 — containerd Full Removal

```bash
# Stop and disable containerd
sudo systemctl stop containerd
sudo systemctl disable containerd

# Kill any stray shim processes
sudo pkill -9 containerd-shim 2>/dev/null || true
sudo pkill -9 containerd      2>/dev/null || true

# Unmount any lingering overlay/container mounts
sudo umount $(mount | grep '/run/containerd' | awk '{print $3}') 2>/dev/null || true
sudo umount $(mount | grep '/var/lib/containerd' | awk '{print $3}') 2>/dev/null || true

# Remove package
sudo apt-get remove --purge -y containerd containerd.io
sudo apt-get autoremove -y

# Remove all data and config
sudo rm -rf /etc/containerd/
sudo rm -rf /var/lib/containerd/
sudo rm -rf /run/containerd/
sudo rm -rf /var/run/containerd/
```

---

## Phase 4 — Flush Network State

```bash
# Flush iptables (legacy and nft)
sudo iptables -F
sudo iptables -X
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo iptables -t mangle -F
sudo iptables -t mangle -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT

sudo ip6tables -F
sudo ip6tables -X
sudo ip6tables -t nat -F
sudo ip6tables -t nat -X
sudo ip6tables -P INPUT ACCEPT
sudo ip6tables -P FORWARD ACCEPT
sudo ip6tables -P OUTPUT ACCEPT

# If nftables is in use
sudo nft flush ruleset 2>/dev/null || true

# Remove virtual interfaces left by Kubernetes/Cilium
for iface in flannel.1 cni0 tunl0 kube-ipvs0 nodelocaldns; do
  sudo ip link delete "$iface" 2>/dev/null || true
done

# Flush any leftover routes injected by CNI
sudo ip route flush proto bird 2>/dev/null || true
sudo ip route flush table local 2>/dev/null || true || true
```

---

## Phase 5 — Remove Residual Kubernetes Data & Config

```bash
# etcd data (control plane)
sudo rm -rf /var/lib/etcd/

# Kubelet state and config
sudo rm -rf /var/lib/kubelet/
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/log/kubernetes/  2>/dev/null || true

# kubeconfig for current user and root
rm -rf "$HOME/.kube/"
sudo rm -rf /root/.kube/

# Helm (if installed and you want clean slate)
# rm -rf "$HOME/.config/helm" "$HOME/.cache/helm"
```

---

## Phase 6 — Kernel Module Cleanup

```bash
# Unload CNI/k8s kernel modules (non-critical if already unloaded)
for mod in br_netfilter overlay ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack; do
  sudo modprobe -r "$mod" 2>/dev/null || true
done

# Remove any persistent module load configs you may have added
sudo rm -f /etc/modules-load.d/k8s.conf
sudo rm -f /etc/modules-load.d/containerd.conf

# Remove sysctl tuning for k8s
sudo rm -f /etc/sysctl.d/99-kubernetes.conf
sudo rm -f /etc/sysctl.d/k8s.conf
sudo sysctl --system   # reload sysctl to revert
```

---

## Phase 7 — APT Repo Cleanup

```bash
# Remove Kubernetes apt repo
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /etc/apt/sources.list.d/docker.list  # if added for containerd
sudo rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo rm -f /etc/apt/trusted.gpg.d/docker.gpg 2>/dev/null || true
sudo apt-get update
```

---

## Phase 8 — Verify Clean State

```bash
# No k8s processes should be running
ps aux | grep -E 'kube|containerd|etcd|cilium' | grep -v grep

# No leftover virtual interfaces
ip link show | grep -E 'cilium|cni|flannel|vxlan|tunl|lxc'

# No stale mounts
mount | grep -E 'containerd|kubelet|overlay'

# No leftover iptables rules
sudo iptables -L -n --line-numbers
sudo iptables -t nat -L -n --line-numbers

# Confirm packages removed
dpkg -l | grep -E 'kube|containerd|cilium'
```

All commands above should return empty output — that confirms a clean node.

---

## Phase 9 — Optional: Snapshot Before Reinstall

Since you're on VMware, **take a snapshot now** before reinstalling:

```
VMware Workstation/ESXi → Right-click VM → Take Snapshot → "clean-base-pre-k8s"
```

This gives you a rollback point if the new install hits issues again.

---

## Architecture View

```
BEFORE (dirty state)               AFTER (clean state)
┌──────────────────────────┐       ┌──────────────────────────┐
│  Ubuntu Guest (VMware)   │       │  Ubuntu Guest (VMware)   │
│  ├─ kube-apiserver       │  ──►  │  ├─ OS + kernel (clean)  │
│  ├─ etcd                 │       │  ├─ apt repos (k8s gone)  │
│  ├─ kubelet              │       │  ├─ iptables: ACCEPT all  │
│  ├─ containerd shims     │       │  ├─ no virtual NICs       │
│  ├─ cilium-agent         │       │  └─ no CNI state          │
│  ├─ lxcXXXXXX (veth)    │       └──────────────────────────┘
│  ├─ cilium_host (NIC)    │
│  └─ iptables (k8s rules) │
└──────────────────────────┘
```

---

## Next 3 Steps

1. **Run the verify block (Phase 8)** — confirm zero residual processes, interfaces, mounts, and iptables rules before touching anything new.
2. **Take a VMware snapshot** — label it `clean-base-pre-k8s`; this is your rollback point.
3. **Plan the reinstall carefully** — before running `kubeadm init` again, share what broke (CNI init failure? kubelet CrashLoop? Cilium pod pending?) so the new setup avoids the same pitfall.

## Summary

Node is **95% clean** — no processes, no mounts, no iptables rules, no packages. Two issues remain: **DNS resolution is broken** (blocks everything) and `tunl0` interface is still present. Fix DNS first — it's the critical blocker. The DNS failure on a VMware guest after a k8s teardown is almost always a broken `/etc/resolv.conf` or a dead `systemd-resolved` socket that k8s networking stomped on.

---

## Step 1 — Diagnose DNS State

```bash
# Check what resolv.conf actually is (symlink vs file)
ls -la /etc/resolv.conf

# Check its content
cat /etc/resolv.conf

# Check systemd-resolved status
sudo systemctl status systemd-resolved

# Check if the stub resolver socket exists
ls -la /run/systemd/resolve/stub-resolv.conf

# Check current DNS from networkd/netplan perspective
resolvectl status

# Check your active NIC and if it got an IP via DHCP
ip addr show
ip route show
```

---

## Step 2 — Fix DNS (apply whichever case matches)

**Case A — resolv.conf is a broken symlink or empty:**
```bash
# Re-link to systemd-resolved stub
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# Restart resolved
sudo systemctl restart systemd-resolved
sudo systemctl enable systemd-resolved
```

**Case B — systemd-resolved is dead:**
```bash
sudo systemctl restart systemd-resolved
sudo systemctl enable systemd-resolved

# Verify it's listening on 127.0.0.53
ss -tulnp | grep 53
```

**Case C — NIC has no IP (DHCP not working after k8s teardown):**
```bash
# Identify your VMware NIC (typically ens33 or ens32)
ip addr show

# Force DHCP renewal
sudo dhclient -v ens33       # replace ens33 with your interface name

# Or via networkd if using netplan
sudo netplan apply
```

**Case D — resolv.conf has wrong/dead nameserver hardcoded:**
```bash
# Temporarily override with a known-good resolver to unblock you
sudo tee /etc/resolv.conf <<EOF
nameserver 8.8.8.8
nameserver 1.1.1.1
search localdomain
EOF

# Then restore the proper symlink after verified working
sudo apt-get update   # test
```

---

## Step 3 — Remove tunl0

```bash
# tunl0 is an IPIP tunnel left by old CNI - remove it
sudo modprobe -r ipip 2>/dev/null || true
sudo ip link set tunl0 down 2>/dev/null || true
sudo ip link delete tunl0 2>/dev/null || true

# Verify gone
ip link show | grep tunl
```

> If `ip link delete tunl0` returns "Cannot remove" — it's owned by the `ipip` kernel module. The `modprobe -r ipip` above handles it. A reboot also clears it cleanly.

---

## Step 4 — Verify Full Clean State

```bash
# DNS works
ping -c 2 google.com
sudo apt-get update

# tunl0 gone
ip link show | grep -E 'tunl|cilium|cni|flannel|lxc'

# Network stack healthy
ip addr show
ip route show
resolvectl status
```

---

## Expected Clean Output

```
# ip link - only real interfaces should remain
1: lo: <LOOPBACK>
2: ens33: <BROADCAST,MULTICAST,UP>   ← your VMware NIC, name may vary

# ping
PING google.com: 64 bytes from ... ttl=... time=... ms

# apt-get update
Hit:1 http://in.archive.ubuntu.com/ubuntu noble InRelease   ← no Err/Ign
```

---

## Next 3 Steps

1. **Run Step 1 diagnostics first** — paste `ls -la /etc/resolv.conf` and `resolvectl status` output so we can confirm the exact DNS failure cause before applying the fix.
2. **After DNS is confirmed working**, take a VMware snapshot: `clean-base-pre-k8s`.
3. **Share what broke in the previous setup** (CNI init? kubelet not starting? Cilium pods stuck?) — so the reinstall is structured to avoid the same failure.

## Summary

Clean, production-grade install of **containerd → Kubernetes 1.32 → Cilium 1.17** on Ubuntu 24.04 (Noble) VMware guest control-plane node. Order matters strictly: kernel/sysctl → containerd → kubeadm/kubelet/kubectl → `kubeadm init` (no kube-proxy, Cilium replaces it) → Cilium via Helm. Security-first at every layer.

---

## Phase 1 — System Prerequisites

```bash
# Disable swap permanently (kubelet requirement)
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
free -h   # verify Swap: 0

# Load required kernel modules
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Verify loaded
lsmod | grep -E 'overlay|br_netfilter'

# Kernel parameters — required for k8s networking
sudo tee /etc/sysctl.d/99-k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
# Verify
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
```

---

## Phase 2 — containerd

```bash
# Install dependencies
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg apt-transport-https

# Docker/containerd apt repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update
sudo apt-get install -y containerd.io

# Generate default config and apply critical changes
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# CRITICAL: Switch cgroup driver to systemd (required for k8s 1.22+)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

# Verify the change
grep 'SystemdCgroup' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd   # must be: active (running)

# Smoke test
sudo ctr version
```

---

## Phase 3 — kubeadm / kubelet / kubectl

```bash
# Kubernetes apt repo (versioned — required for 1.32)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # prevent unintended upgrades

# Verify
kubeadm version
kubectl version --client
kubelet --version
```

---

## Phase 4 — kubeadm init (Control Plane)

```bash
# Get your VMware NIC IP (typically ens33/ens32)
ip addr show | grep 'inet ' | grep -v '127.0.0.1'
# Note down your node IP — replace 192.168.x.x below

# IMPORTANT: --skip-phases=addon/kube-proxy
# Cilium runs in kube-proxy replacement mode (eBPF datapath)
# Do NOT install kube-proxy — it will conflict with Cilium

sudo kubeadm init \
  --apiserver-advertise-address=<YOUR_NODE_IP> \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --skip-phases=addon/kube-proxy \
  2>&1 | tee ~/kubeadm-init.log

# Save the join command printed at the end — you'll need it for workers
```

```bash
# Setup kubeconfig for your user
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"

# Verify API server is up
kubectl get nodes        # STATUS will be NotReady — expected, no CNI yet
kubectl get pods -A      # coredns pods will be Pending — expected
```

---

## Phase 5 — Cilium via Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \
  | bash
helm version

# Add Cilium repo
helm repo add cilium https://helm.cilium.io/
helm repo update

# Install Cilium with kube-proxy replacement (eBPF mode)
# KUBECONFIG must be set
helm install cilium cilium/cilium \
  --version 1.17.3 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<YOUR_NODE_IP> \
  --set k8sServicePort=6443 \
  --set ipam.mode=kubernetes \
  --set tunnel=vxlan \
  --set image.pullPolicy=IfNotPresent \
  --set operator.replicas=1

# Watch rollout
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system rollout status deploy/cilium-operator
```

---

## Phase 6 — Verify Full Stack

```bash
# Node must be Ready now
kubectl get nodes -o wide

# All system pods Running
kubectl get pods -A

# Cilium CLI for deeper validation
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz"
sudo tar -C /usr/local/bin -xzf cilium-linux-amd64.tar.gz
rm cilium-linux-amd64.tar.gz

# Full Cilium connectivity test
cilium status --wait
cilium connectivity test    # runs ~50 end-to-end network tests
```

---

## Architecture View

```
Ubuntu 24.04 Guest (VMware)
┌──────────────────────────────────────────────────────┐
│  Control Plane                                        │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────────┐ │
│  │ kube-        │  │ etcd     │  │ kube-controller │ │
│  │ apiserver    │  │ :2379    │  │ kube-scheduler  │ │
│  └──────┬──────┘  └──────────┘  └─────────────────┘ │
│         │                                             │
│  ┌──────▼──────────────────────────────────────────┐ │
│  │  kubelet  (cgroup driver: systemd)               │ │
│  │    └── containerd (SystemdCgroup=true)           │ │
│  │          └── containerd-shim-runc-v2             │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │  Cilium (kube-proxy replacement, eBPF)           │ │
│  │  ├── cilium-agent   (DaemonSet)                  │ │
│  │  ├── cilium-operator                             │ │
│  │  ├── eBPF datapath  (replaces iptables/kube-proxy│ │
│  │  └── IPAM: pod CIDR 10.244.0.0/16               │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  Kernel: overlay + br_netfilter + ip_forward=1        │
│  Cgroups: systemd (unified v2)                        │
└──────────────────────────────────────────────────────┘
         │ VMware VMXNET3
    Host Network / NAT / Bridged
```

---

## Threat Model (Single-Node Lab Context)

| Threat | Mitigation |
|---|---|
| Container escape via runc | containerd + runc with `SystemdCgroup=true`, no privileged pods |
| CNI bypass / pod spoofing | Cilium eBPF replaces iptables — kernel-enforced datapath |
| Unauthenticated API access | kubeadm generates TLS certs + RBAC enabled by default |
| etcd exposure | Bound to localhost only by kubeadm default |
| Swap leaking secrets | Swap disabled before kubelet starts |

---

## Rollback Plan

```bash
# If kubeadm init fails mid-way
sudo kubeadm reset --force --cri-socket unix:///run/containerd/containerd.sock
rm -rf "$HOME/.kube"

# If Cilium install fails, reinstall cleanly
helm uninstall cilium -n kube-system
kubectl delete namespace cilium-test 2>/dev/null || true
# Then re-run Phase 5
```

If anything goes wrong before Cilium, the VMware snapshot (`clean-base-pre-k8s`) is your hard rollback.

---

## Next 3 Steps

1. **Run Phase 1–2 now**, then paste `sudo systemctl status containerd` — confirm `SystemdCgroup = true` is set before touching kubeadm.
2. **Confirm your node IP** with `ip addr show` and decide: NAT or bridged networking in VMware — this affects `--apiserver-advertise-address` and whether worker nodes can reach the API server later.
3. **After `cilium connectivity test` passes**, share the output — we'll then harden the setup with `NetworkPolicy`, audit logging, and `seccompProfile` defaults.

Quick fix — `tunnel` was replaced in v1.15 with `routingMode` + `tunnelProtocol`.

```bash
helm install cilium cilium/cilium \
  --version 1.17.3 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=172.16.151.129 \
  --set k8sServicePort=6443 \
  --set ipam.mode=kubernetes \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set image.pullPolicy=IfNotPresent \
  --set operator.replicas=1
```

Then watch the rollout:

```bash
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system rollout status deploy/cilium-operator
kubectl get pods -n kube-system
```

**Why this works:** Cilium 1.15+ unified tunnel config into `routingMode=tunnel` (encap mode) vs `routingMode=native` (direct routing). `tunnelProtocol` then selects `vxlan` or `geneve`. On a VMware guest, `vxlan` is the right choice — `geneve` requires NIC offload support that VMXNET3 doesn't reliably provide.

Once pods are Running, paste `cilium status --wait` output and we'll confirm the eBPF datapath is active.

Cilium CLI isn't installed yet. Install it:

```bash
# Fetch latest stable version tag
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)

# Download for amd64
curl -L --fail \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz" \
  -o /tmp/cilium.tar.gz

# Install to PATH
sudo tar -C /usr/local/bin -xzf /tmp/cilium.tar.gz
rm /tmp/cilium.tar.gz

# Verify
cilium version
```

Then check status:

```bash
cilium status --wait
```

Paste the output — specifically looking for:
- `KubeProxyReplacement: True`
- `cilium-operator: OK`
- `Cluster health: N/N reachable`

If the GitHub download is slow/blocked, alternative direct install:

```bash
# Check what version is available
curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt

# Or install via apt if you prefer (older but works)
# Not recommended — use the binary above for version control
```

Yes — Hubble is built into Cilium, just needs enabling via Helm upgrade. No separate install needed.

## Enable Hubble + Hubble UI

```bash
helm upgrade cilium cilium/cilium \
  --version 1.17.3 \
  --namespace kube-system \
  --reuse-values \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.ui.service.type=NodePort
```

## Watch Rollout

```bash
kubectl -n kube-system rollout status deploy/hubble-relay
kubectl -n kube-system rollout status deploy/hubble-ui
kubectl get pods -n kube-system | grep hubble
```

## Access Hubble UI

```bash
# Get the NodePort assigned
kubectl get svc -n kube-system hubble-ui

# Output will show something like:
# hubble-ui   NodePort   10.x.x.x   <none>   80:31234/TCP
# Access via: http://172.16.151.129:<NodePort>
```

Then open in your browser: `http://172.16.151.129:<NodePort>`

## Verify Hubble via CLI

```bash
# Hubble CLI is separate from cilium CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail \
  "https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz" \
  -o /tmp/hubble.tar.gz
sudo tar -C /usr/local/bin -xzf /tmp/hubble.tar.gz
rm /tmp/hubble.tar.gz

hubble version

# Port-forward Hubble relay and observe live flows
kubectl port-forward -n kube-system svc/hubble-relay 4245:80 &
hubble observe --follow
```

Paste `kubectl get pods -n kube-system | grep hubble` output once done — we'll confirm all three components (relay, ui, peer-service) are Running.

Pods are **Pending** — not crashing, not pulling. This means a scheduling problem: taints, resource constraints, or image pull issues. Run these in order:

## Step 1 — Describe the Pending Pods

```bash
# Get exact pod names first
kubectl get pods -n kube-system | grep hubble

# Describe each pending pod — the Events section at the bottom is key
kubectl describe pod -n kube-system -l k8s-app=hubble-relay
kubectl describe pod -n kube-system -l k8s-app=hubble-ui
```

**Look at the bottom `Events:` section** — paste that output. Most likely culprit:

---

## Step 2 — Check Node Taints (Most Common Cause)

```bash
# Control-plane nodes have a NoSchedule taint by default
# kubeadm adds: node-role.kubernetes.io/control-plane:NoSchedule
kubectl describe node | grep -A5 Taints
```

If you see `node-role.kubernetes.io/control-plane:NoSchedule` — that's the problem. This is a **single-node cluster**, so you must untaint it:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
# The trailing `-` removes the taint

# Verify taint is gone
kubectl describe node | grep Taint
```

Pods should start scheduling within 30 seconds after this.

---

## Step 3 — Check Node Resources

```bash
# Is the node under memory/cpu pressure?
kubectl describe node | grep -A10 "Conditions:"
kubectl describe node | grep -A10 "Allocated resources"

# Quick resource check
free -h
df -h /var/lib/containerd   # disk space for image pulls
```

---

## Step 4 — Check Image Pull Status

```bash
# Are images being pulled or stuck?
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep -E 'hubble|Pull|Failed|Back'

# Check containerd is pulling
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock images | grep hubble
```

---

## Step 5 — Watch Recovery After Untaint

```bash
# After removing the taint, watch pods come up
kubectl get pods -n kube-system -w | grep hubble

# Once Running, verify
cilium status --wait
```

---

**Paste the output of Step 1 (`kubectl describe pod`) and Step 2 (taint check)** — 90% chance it's the control-plane taint on a single-node setup.

## Browser Access

```bash
# Check the NodePort assigned
kubectl get svc -n kube-system hubble-ui
# Look for: 80:<NodePort>/TCP
```

Open browser on your **host machine** (not the VM):
```
http://172.16.151.129:<NodePort>
```

If NodePort isn't convenient, use port-forward instead:
```bash
# Forward to 0.0.0.0 so your host browser can reach it
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 --address=0.0.0.0 &

# Access at:
# http://172.16.151.129:12000
```

---

## Terminal Access (Hubble CLI)

```bash
# Install Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail \
  "https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz" \
  -o /tmp/hubble.tar.gz
sudo tar -C /usr/local/bin -xzf /tmp/hubble.tar.gz
rm /tmp/hubble.tar.gz

hubble version
```

```bash
# Port-forward Hubble relay (required for CLI to talk to it)
kubectl port-forward -n kube-system svc/hubble-relay 4245:80 &

# Verify relay connection
hubble status --server localhost:4245

# Observe all live flows
hubble observe --server localhost:4245 --follow

# Filter by namespace
hubble observe --server localhost:4245 --namespace kube-system --follow

# Filter by verdict (dropped packets — useful for policy debugging)
hubble observe --server localhost:4245 --verdict DROPPED --follow

# Filter by specific pod
hubble observe --server localhost:4245 --pod hubble-relay --follow
```

---

## Persist Port-Forwards (Optional)

Instead of `&` backgrounding, run them in a dedicated terminal or as a simple service:

```bash
# Both forwards in one shot
kubectl port-forward -n kube-system svc/hubble-ui    12000:80  --address=0.0.0.0 &
kubectl port-forward -n kube-system svc/hubble-relay 4245:80   --address=0.0.0.0 &

# See background jobs
jobs

# Kill all port-forwards when done
pkill -f "kubectl port-forward"
```

---

## Quick Smoke Test

```bash
# Generate some traffic to see flows in UI/CLI
kubectl run test-pod --image=nginx --restart=Never
kubectl get pod test-pod

# Watch flows appear in real time
hubble observe --server localhost:4245 --follow
```

---

## Next 3 Steps

1. **Open `http://172.16.151.129:<NodePort>`** in your host browser — select `kube-system` namespace in the dropdown to see live flows immediately.
2. **Run `hubble observe --verdict DROPPED --follow`** — on a fresh cluster this should be silent (no drops), confirming no unintended NetworkPolicy blocking.
3. **Deploy a test workload** (nginx + curl pod) and watch the flows — this validates the full Cilium eBPF datapath end-to-end before we add `NetworkPolicy` rules.

