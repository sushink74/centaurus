## Summary

Raspberry Pi 4 (ARM64/aarch64) makes a capable Kubernetes data-plane node. You'll flash a 64-bit OS, harden the kernel, install a CRI (containerd is the production choice; CRI-O is solid too), bootstrap kubeadm, and join the node as a worker. Key gotchas: cgroups v2 must be forced via cmdline, swap must be off, and iptables must use legacy mode on older kernels. This guide is production-minded — you can run this in a home-lab or edge cluster.

---

## Architecture

```
 Control Plane (x86 or another Pi)
 ┌─────────────────────────────────┐
 │  kube-apiserver                 │
 │  kube-scheduler                 │
 │  kube-controller-manager        │
 │  etcd                           │
 └──────────────┬──────────────────┘
                │ kubeadm join token / mTLS
 ┌──────────────▼──────────────────┐
 │   Raspberry Pi 4 (Data Plane)   │
 │                                 │
 │  kubelet ──► CRI socket         │
 │              │                  │
 │   containerd / CRI-O            │
 │              │                  │
 │   runc / crun (OCI runtime)     │
 │              │                  │
 │   Pod sandbox (pause container) │
 │   ├── app container             │
 │   └── sidecar container         │
 │                                 │
 │  kube-proxy  (iptables/ipvs)    │
 │  CNI plugin  (Flannel/Calico)   │
 └─────────────────────────────────┘
       ARM64 / aarch64 / Linux 6.x
```

---

## Step 0 — Flash 64-bit OS

Use **Ubuntu Server 24.04 LTS ARM64** (preferred for cgroups v2 out of box) or Raspberry Pi OS 64-bit Lite.

```bash
# On your workstation
wget https://cdimage.ubuntu.com/releases/24.04/release/ubuntu-24.04.2-preinstalled-server-arm64+raspi.img.xz
xz -d ubuntu-24.04.2-preinstalled-server-arm64+raspi.img.xz
sudo dd if=ubuntu-24.04.2-preinstalled-server-arm64+raspi.img of=/dev/sdX bs=4M status=progress conv=fsync
```

---

## Step 1 — OS Hardening & Kernel Prep

```bash
# SSH in, then:
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget gnupg2 apt-transport-https ca-certificates \
     socat conntrack ipset ipvsadm ethtool

# Set hostname
sudo hostnamectl set-hostname pi4-worker-01

# Disable swap permanently
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
# Verify
free -h   # Swap line must show 0

# Force cgroups v2 + enable memory cgroup
sudo sed -i 's/^/cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 systemd.unified_cgroup_hierarchy=1 /' \
    /boot/firmware/cmdline.txt
# Ubuntu 24.04 path is /boot/firmware/cmdline.txt
# Raspberry Pi OS path is /boot/cmdline.txt — adjust accordingly

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

sudo modprobe overlay br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack

# Sysctl
cat <<EOF | sudo tee /etc/sysctl.d/99-k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv6.conf.all.forwarding        = 1
fs.inotify.max_user_watches         = 524288
fs.inotify.max_user_instances       = 512
vm.overcommit_memory                = 1
kernel.panic                        = 10
kernel.panic_on_oops                = 1
EOF
sudo sysctl --system

sudo reboot
```

After reboot, verify cgroups:

```bash
cat /proc/cgroups | grep memory   # non-zero hierarchy
ls /sys/fs/cgroup/                # unified v2 has no "memory" subdir, just files
```

---

## Step 2 — Install containerd (Recommended CRI)

```bash
# Install containerd from Docker's repo (always has latest ARM64 builds)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y containerd.io

# Generate default config and enable cri plugin (disabled by default in some builds)
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Critical: set systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
    /etc/containerd/config.toml

# Verify the sandbox (pause) image — use a versioned one
grep sandbox_image /etc/containerd/config.toml
# Should show: registry.k8s.io/pause:3.10 (or latest)

sudo systemctl enable --now containerd
sudo systemctl status containerd

# Smoke test
sudo ctr version
sudo ctr images pull docker.io/library/hello-world:latest
sudo ctr run --rm docker.io/library/hello-world:latest test
```

---

## Step 2 (alt) — Install CRI-O instead

```bash
# CRI-O version must match Kubernetes minor version
KUBE_VERSION=v1.32
OS=xUbuntu_24.04

cat <<EOF | sudo tee /etc/apt/sources.list.d/crio.list
deb https://pkgs.k8s.io/addons:/cri-o:/stable:/${KUBE_VERSION}/deb/ /
EOF

curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/${KUBE_VERSION}/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

sudo apt update && sudo apt install -y cri-o

sudo systemctl enable --now crio
sudo systemctl status crio

# CRI-O uses systemd cgroup by default — verify
grep cgroup /etc/crio/crio.conf
```

---

## Step 2 (opt) — Docker Engine (if you need it alongside containerd)

```bash
# Docker is NOT a CRI — kubelet talks to containerd socket, not dockerd
# Install only if you need docker CLI for image builds on the node
sudo apt install -y docker-ce docker-ce-cli

# containerd.sock is still the CRI endpoint
# Docker uses its own containerd shim — they share /run/containerd/containerd.sock
```

---

## Step 3 — Install kubeadm, kubelet, kubectl

```bash
KUBE_VERSION=1.32

curl -fsSL https://pkgs.k8s.io/core:/stable:/v${KUBE_VERSION}/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v${KUBE_VERSION}/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Tell kubelet which CRI socket to use (containerd)
cat <<EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS=--container-runtime-endpoint=unix:///run/containerd/containerd.sock
EOF

# For CRI-O use: unix:///var/run/crio/crio.sock

sudo systemctl enable kubelet
```

---

## Step 4 — Join as Data Plane (Worker) Node

On your **control plane**, generate a join command:

```bash
# On control plane
kubeadm token create --print-join-command
# Output: kubeadm join <CP_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

On the **Pi**:

```bash
sudo kubeadm join <CP_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///run/containerd/containerd.sock

# If joining fails, reset and retry
sudo kubeadm reset --cri-socket unix:///run/containerd/containerd.sock
```

Verify from control plane:

```bash
kubectl get nodes -o wide
# Pi should show: Ready   <none>  worker
kubectl describe node pi4-worker-01 | grep -A5 "Capacity\|Allocatable"
```

---

## Step 5 — CNI Plugin (pick one)

**Flannel** (simplest, ARM64 supported):

```bash
# On control plane (if not already applied)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**Calico** (production, supports NetworkPolicy):

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml
# Then apply custom-resources.yaml with your pod CIDR
```

---

## Step 6 — Label Node as Data Plane

```bash
kubectl label node pi4-worker-01 node-role.kubernetes.io/worker=worker
kubectl label node pi4-worker-01 kubernetes.io/arch=arm64
kubectl label node pi4-worker-01 topology.kubernetes.io/zone=edge-rack-01

# Taint if you want only specific workloads here
kubectl taint node pi4-worker-01 dedicated=edge:NoSchedule
```

---

## Threat Model & Mitigations

| Threat | Mitigation |
|---|---|
| Unauthorized kubelet API access | `--anonymous-auth=false`, rotate bootstrap tokens, mTLS enforced by kubeadm |
| Container escape via runc CVEs | Pin containerd+runc versions, enable seccomp defaults, use `--privileged=false` |
| Node compromise via physical access | Full-disk encryption (dm-crypt), disable USB boot, lock EEPROM boot order |
| Lateral movement via pod networking | NetworkPolicy + Calico, disable host networking for workloads |
| Image supply chain | Use image digest pinning, enable containerd image verification (cosign/notation) |
| Privileged pods | OPA/Gatekeeper or Kyverno to deny `hostPID`, `hostNetwork`, `privileged` |
| Token theft | Bound service account tokens (projected volumes), short TTLs |

---

## Verify & Test

```bash
# Confirm CRI is healthy
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock info
sudo crictl ps

# Run a test pod targeting the Pi
kubectl run test-arm --image=arm64v8/alpine --overrides='{"spec":{"nodeSelector":{"kubernetes.io/arch":"arm64"}}}' \
  --command -- sleep 3600

kubectl exec -it test-arm -- uname -m   # should output: aarch64

# Check resource pressure
kubectl top node pi4-worker-01   # requires metrics-server
kubectl describe node pi4-worker-01 | grep -A10 "Conditions:"

# containerd performance
sudo ctr plugins ls
sudo journalctl -u containerd -f
```

---

## Roll-out / Rollback Plan

```bash
# Drain node before any upgrades
kubectl drain pi4-worker-01 --ignore-daemonsets --delete-emptydir-data

# Upgrade containerd
sudo apt install -y containerd.io=<new-version>
sudo systemctl restart containerd

# Upgrade kubelet/kubeadm
sudo apt-mark unhold kubelet kubeadm
sudo apt install -y kubelet=1.33.0-1.1 kubeadm=1.33.0-1.1
sudo apt-mark hold kubelet kubeadm
sudo systemctl restart kubelet

# Uncordon after validation
kubectl uncordon pi4-worker-01

# Full rollback: reset and rejoin
sudo kubeadm reset --cri-socket unix:///run/containerd/containerd.sock
sudo iptables -F && sudo iptables -X
sudo ipvsadm --clear
# Re-run join command
```

---

## References

- [kubeadm install docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [containerd CRI setup](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)
- [CRI-O on Kubernetes](https://cri-o.io/)
- [Flannel ARM64](https://github.com/flannel-io/flannel)
- [Calico on ARM](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [Pi 4 cgroups v2](https://wiki.ubuntu.com/ARM/RaspberryPi)

---

## Next 3 Steps

1. **Deploy metrics-server** (`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`) and add `--kubelet-insecure-tls` arg for home-lab, then baseline CPU/memory for Pi workloads.
2. **Harden kubelet config** — create `/etc/kubernetes/kubelet-config.yaml` with `rotateCertificates: true`, `protectKernelDefaults: true`, `eventRecordQPS: 5`, and apply CIS benchmark checks via `kube-bench`.
3. **Enable AppArmor/seccomp profiles** on the node — verify `cat /sys/module/apparmor/parameters/enabled`, then enforce the `RuntimeDefault` seccomp profile cluster-wide via a Kyverno policy so all pods on the Pi get syscall filtering without per-pod annotation.