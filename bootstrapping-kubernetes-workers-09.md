# Section 9: Bootstrapping Kubernetes Worker Nodes

## Overview

This section bootstraps the two worker nodes where your application pods will actually run. While the control plane (Section 8) manages the cluster, worker nodes do the real work of running containers.

## Key Components

### 1. **containerd** - The Container Runtime

**What it is:** The engine that actually runs containers (like Docker, but simpler and more focused)

**Analogy:** If Kubernetes is a shipping port manager, containerd is the crane operator that physically moves containers

**Why:** Kubernetes doesn't run containers itself - it delegates to a container runtime via the CRI (Container Runtime Interface)

**Config:** `/etc/containerd/config.toml` - tells containerd how to manage containers, use systemd cgroups, etc.

### 2. **kubelet** - The Node Agent

**What it is:** The primary agent on each worker node that talks to the API server

**Analogy:** Like a construction foreman who receives blueprints (pod specs) from headquarters (API server) and ensures workers (containers) build according to plan

**What it does:**

- Registers the node with the cluster
- Watches the API server for pods assigned to this node
- Tells containerd to start/stop containers
- Reports node and pod status back to API server
- Mounts volumes, manages secrets

**Config:** `/var/lib/kubelet/kubelet-config.yaml` - node-specific settings like cluster DNS, pod CIDR

**Certificates needed:**

- `/var/lib/kubelet/ca.crt` - CA certificate to verify API server
- `/var/lib/kubelet/kubelet.crt` - Node's client certificate (node-0.crt or node-1.crt)
- `/var/lib/kubelet/kubelet.key` - Node's private key (node-0.key or node-1.key)

### 3. **kube-proxy** - The Network Proxy

**What it is:** Manages network rules on each node for service networking

**Analogy:** Like a switchboard operator routing calls - ensures traffic to a Service IP gets routed to the right pod

**What it does:**

- Watches the API server for Service and Endpoint changes
- Updates iptables/IPVS rules to route service traffic to pods
- Enables the Service abstraction (stable IP for dynamic pods)

**Config:** `/var/lib/kube-proxy/kube-proxy-config.yaml` - cluster CIDR, proxy mode

### 4. **CNI Plugins** - Container Network Interface

**What it is:** Plugins that set up pod networking

**Analogy:** Like electricians who wire up each apartment (pod) to the building's network infrastructure

**What they do:**

- Create network interfaces for pods
- Assign IP addresses from the pod CIDR
- Set up routing between pods
- Configure bridge networks on the host

**Plugins installed:**

- `bridge` - Creates a network bridge for pods
- `loopback` - Sets up localhost (127.0.0.1) inside containers
- `host-local` - Assigns IP addresses from a range (IPAM)

**Configs:**

- `/etc/cni/net.d/10-bridge.conf` - Node-specific subnet (10.200.0.0/24 or 10.200.1.0/24)
- `/etc/cni/net.d/99-loopback.conf` - Loopback interface config

## Architecture Flow

```text
┌─────────────────────────────────────────────────┐
│             Control Plane (server)              │
│          kube-apiserver ← kubelet talks here    │
└────────────────────┬────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
┌────────▼─────────┐    ┌────────▼─────────┐
│  Worker: node-0  │    │  Worker: node-1  │
│  10.240.0.20     │    │  10.240.0.21     │
├──────────────────┤    ├──────────────────┤
│  kubelet         │    │  kubelet         │
│  ├─ registers    │    │  ├─ registers    │
│  ├─ watches      │    │  ├─ watches      │
│  └─ reports      │    │  └─ reports      │
├──────────────────┤    ├──────────────────┤
│  containerd      │    │  containerd      │
│  └─ runs pods    │    │  └─ runs pods    │
├──────────────────┤    ├──────────────────┤
│  kube-proxy      │    │  kube-proxy      │
│  └─ routes svcs  │    │  └─ routes svcs  │
├──────────────────┤    ├──────────────────┤
│  CNI plugins     │    │  CNI plugins     │
│  Pod CIDR:       │    │  Pod CIDR:       │
│  10.200.0.0/24   │    │  10.200.1.0/24   │
└──────────────────┘    └──────────────────┘
```

## Setup Process

### Dependencies Installation

```bash
apt-get update
apt-get -y install socat conntrack ipset kmod
```

**Why each package:**

- `socat` - Port forwarding (kubectl port-forward uses this)
- `conntrack` - Connection tracking for kube-proxy's iptables rules
- `ipset` - Efficient IP address matching for network policies
- `kmod` - Kernel module loading (needed for `br-netfilter`)

### Disable Swap

```bash
swapon --show  # Should be empty
```

**Why:** Kubernetes requires swap disabled for predictable pod memory management

### Directory Structure

```bash
mkdir -p   /etc/cni/net.d \           # CNI network configs
  /opt/cni/bin \             # CNI plugin binaries
  /var/lib/kubelet \         # Kubelet data and certs
  /var/lib/kube-proxy \      # Kube-proxy config
  /var/lib/kubernetes \      # General k8s data
  /var/run/kubernetes        # Runtime files
```

### Binary Installation

Binaries are distributed from the jumpbox and moved to:

- `/usr/local/bin/` - kubelet, kube-proxy, crictl, runc
- `/bin/` - containerd and shim binaries
- `/opt/cni/bin/` - CNI plugins

### Network Configuration

```bash
modprobe br-netfilter
echo "br-netfilter" >> /etc/modules-load.d/modules.conf
```

**Why:** Enables the kernel module for bridging and filtering - required for pod networking

```bash
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/kubernetes.conf
sysctl -p /etc/sysctl.d/kubernetes.conf
```

**Why:** Allows iptables to see bridged traffic - critical for service routing and network policies

### Certificate Setup (Critical!)

Each worker node needs three certificates in `/var/lib/kubelet/`:

```bash
cp /root/node-X.crt /var/lib/kubelet/kubelet.crt   # Node identity
cp /root/node-X.key /var/lib/kubelet/kubelet.key   # Private key
cp /root/ca.crt /var/lib/kubelet/ca.crt            # CA for verification
```

**Why the renaming:** kubelet expects generic names (`kubelet.crt`) but you generated node-specific certs (`node-0.crt`, `node-1.crt`) for unique identities

**What happens:**

- kubelet uses its cert to authenticate with kube-apiserver (mTLS)
- kube-apiserver verifies the cert was signed by the CA
- The cert's CN (CommonName) becomes the node's identity

### Service Management

```bash
systemctl daemon-reload                          # Reload service files
systemctl enable containerd kubelet kube-proxy   # Auto-start on boot
systemctl start containerd kubelet kube-proxy    # Start now
```

**Service dependencies:**

1. containerd must start first (kubelet needs it)
2. kubelet registers node and starts pods
3. kube-proxy sets up service routing

## Troubleshooting

### Issue: kubelet stuck in "activating"

**Symptom:** `systemctl is-active kubelet` shows "activating" indefinitely

**Check logs:**

```bash
journalctl -u kubelet --no-pager -n 50
```

**Common causes:**

1. **Missing ca.crt**
   ```text
   Error: unable to load client CA file /var/lib/kubelet/ca.crt
   ```
   **Fix:** Copy from jumpbox: `scp ca.crt root@node-X:/var/lib/kubelet/`

2. **Missing kubelet.crt/kubelet.key**
   ```text
   Error: open /var/lib/kubelet/kubelet.crt: no such file or directory
   ```
   **Fix:** Copy node-specific certs with generic names:
   ```bash
   cp /root/node-X.crt /var/lib/kubelet/kubelet.crt
   cp /root/node-X.key /var/lib/kubelet/kubelet.key
   ```

3. **Can't reach API server**
   Check network connectivity to server (10.240.0.10:6443)

## Verification

### Check Services on Worker Nodes

```bash
# On each worker node
systemctl is-active containerd  # Should show: active
systemctl is-active kubelet     # Should show: active
systemctl is-active kube-proxy  # Should show: active
```

### Verify Node Registration

```bash
# From jumpbox or server
kubectl get nodes --kubeconfig admin.kubeconfig
```

**Expected output:**

```text
NAME     STATUS   ROLES    AGE   VERSION
node-0   Ready    <none>   40m   v1.32.3
node-1   Ready    <none>   54s   v1.32.3
```

**Status meanings:**

- `Ready` - kubelet is healthy and ready to accept pods
- `NotReady` - kubelet is running but not healthy (usually networking issues)
- `<none>` - No roles assigned (normal for worker nodes in this setup)

## How Everything Connects

1. **kubelet** starts and reads its config and certificates
2. kubelet connects to **kube-apiserver** (10.240.0.10:6443) using mTLS
3. kubelet registers the node with the cluster
4. kubelet tells **containerd** it's ready to run containers
5. **kube-proxy** starts and watches the API server for services
6. kube-proxy sets up initial iptables rules
7. **CNI plugins** are ready to configure networking when pods start

**When a pod is scheduled:**

1. kube-apiserver assigns pod to node-0
2. kubelet sees the new pod assignment
3. kubelet asks containerd to pull images
4. kubelet asks CNI to set up pod networking
5. CNI assigns IP from 10.200.0.0/24 range
6. containerd starts the containers
7. kubelet reports pod status back to API server

## Production Differences

**In KTHW (tutorial):**

- Manual certificate management
- No automatic certificate rotation
- Simple CNI setup (bridge plugin)
- Three total nodes (1 control plane, 2 workers)

**In Production:**

- Certificate auto-rotation enabled
- More sophisticated CNI (Calico, Cilium, Weave)
- Dedicated control plane nodes (often 3+ for HA)
- Worker nodes scaled based on workload
- Node pools with different instance types
- Monitoring and logging agents (Prometheus, Fluentd)
- Security agents (Falco, Twistlock)
