# Section 5: Kubernetes Configuration Files (Kubeconfig)

## Overview

Kubeconfig files bundle cluster information, certificates, and user credentials into portable configuration files that Kubernetes components use to connect and authenticate to the API server.

**Restaurant Reservation Analogy:**
- **Kubeconfig file** = Complete reservation confirmation (restaurant address, your name, reservation time)
- **Cluster info** = Restaurant address and phone number
- **User credentials** = Your name and confirmation number
- **Context** = Specific reservation (combines where + who)

Instead of components managing 3 separate files (CA cert, client cert, client key), they load one kubeconfig file with everything embedded.

---

## Kubeconfig Structure

A kubeconfig file has 3 main sections:

```yaml
clusters:           # Where to connect (API server address + CA cert)
users:              # How to authenticate (client cert + key)
contexts:           # Which cluster + which user (profile)
current-context:    # Active profile to use
```

### Why This Approach

- Portable: One file instead of three
- Organized: All connection info in one place
- Standard: All Kubernetes tools understand this format
- Flexible: Can store multiple clusters/users, switch between them

---

## What You'll Create

| Kubeconfig File | Used By | Runs On | Connects As |
|---|---|---|---|
| `node-0.kubeconfig` | kubelet | node-0 | `system:node:node-0` |
| `node-1.kubeconfig` | kubelet | node-1 | `system:node:node-1` |
| `kube-proxy.kubeconfig` | kube-proxy | Both workers | `system:kube-proxy` |
| `kube-controller-manager.kubeconfig` | controller-manager | server | `system:kube-controller-manager` |
| `kube-scheduler.kubeconfig` | scheduler | server | `system:kube-scheduler` |
| `admin.kubeconfig` | kubectl | jumpbox | `admin` |

---

## Step 1: Generate Kubelet Kubeconfig Files (Workers)

**Purpose:** Create kubeconfig files for kubelet on each worker node.

**Why node-specific:** Each kubelet must authenticate with its own node certificate (`node-0` uses `node-0.crt`, `node-1` uses `node-1.crt`) for proper Node Authorization.

```bash
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way     --certificate-authority=ca.crt     --embed-certs=true     --server=https://server.kubernetes.local:6443     --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host}     --client-certificate=${host}.crt     --client-key=${host}.key     --embed-certs=true     --kubeconfig=${host}.kubeconfig

  kubectl config set-context default     --cluster=kubernetes-the-hard-way     --user=system:node:${host}     --kubeconfig=${host}.kubeconfig

  kubectl config use-context default     --kubeconfig=${host}.kubeconfig
done
```

### What each command does

- `set-cluster`: Defines cluster (API server address + CA cert to trust)
- `set-credentials`: Defines user (client cert + key for authentication)
- `set-context`: Links cluster + user together
- `use-context`: Sets as active/default context

**Files created:** `node-0.kubeconfig`, `node-1.kubeconfig`

---

## Step 2: Generate Kube-Proxy Kubeconfig

**Purpose:** Create kubeconfig for kube-proxy service (runs on all workers).

**Why same file for both workers:** kube-proxy uses same identity (`system:kube-proxy`) on all nodes; it doesn't need node-specific identity.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way     --certificate-authority=ca.crt     --embed-certs=true     --server=https://server.kubernetes.local:6443     --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy     --client-certificate=kube-proxy.crt     --client-key=kube-proxy.key     --embed-certs=true     --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default     --cluster=kubernetes-the-hard-way     --user=system:kube-proxy     --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default     --kubeconfig=kube-proxy.kubeconfig
}
```

**File created:** `kube-proxy.kubeconfig`

---

## Step 3: Generate Kube-Controller-Manager Kubeconfig

**Purpose:** Create kubeconfig for controller-manager (runs on server).

**Why needed:** Controller-manager needs to authenticate to API server to watch cluster state and manage controllers.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way     --certificate-authority=ca.crt     --embed-certs=true     --server=https://server.kubernetes.local:6443     --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager     --client-certificate=kube-controller-manager.crt     --client-key=kube-controller-manager.key     --embed-certs=true     --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default     --cluster=kubernetes-the-hard-way     --user=system:kube-controller-manager     --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default     --kubeconfig=kube-controller-manager.kubeconfig
}
```

**File created:** `kube-controller-manager.kubeconfig`

---

## Step 4: Generate Kube-Scheduler Kubeconfig

**Purpose:** Create kubeconfig for scheduler (runs on server).

**Why needed:** Scheduler needs to authenticate to API server to read pod queue and assign pods to nodes.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way     --certificate-authority=ca.crt     --embed-certs=true     --server=https://server.kubernetes.local:6443     --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler     --client-certificate=kube-scheduler.crt     --client-key=kube-scheduler.key     --embed-certs=true     --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default     --cluster=kubernetes-the-hard-way     --user=system:kube-scheduler     --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default     --kubeconfig=kube-scheduler.kubeconfig
}
```

**File created:** `kube-scheduler.kubeconfig`

---

## Step 5: Generate Admin Kubeconfig

**Purpose:** Create kubeconfig for admin user (`kubectl` commands from jumpbox).

**Why different server address:** Uses `127.0.0.1:6443` (localhost) because admin will run `kubectl` from the jumpbox which will proxy to the server. Other components connect directly to `server.kubernetes.local:6443`.

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way     --certificate-authority=ca.crt     --embed-certs=true     --server=https://127.0.0.1:6443     --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin     --client-certificate=admin.crt     --client-key=admin.key     --embed-certs=true     --kubeconfig=admin.kubeconfig

  kubectl config set-context default     --cluster=kubernetes-the-hard-way     --user=admin     --kubeconfig=admin.kubeconfig

  kubectl config use-context default     --kubeconfig=admin.kubeconfig
}
```

**File created:** `admin.kubeconfig`

---

## Step 6: Distribute Kubeconfig Files

**Purpose:** Copy kubeconfig files to the VMs where they'll be used.

### Worker Nodes (node-0, node-1)

**What you're doing:**
- Create directories for kubelet and kube-proxy configs
- Copy `kube-proxy.kubeconfig` to `/var/lib/kube-proxy/kubeconfig` (same file on both)
- Copy node-specific kubelet config to `/var/lib/kubelet/kubeconfig`

```bash
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig     root@${host}:/var/lib/kube-proxy/kubeconfig

  scp ${host}.kubeconfig     root@${host}:/var/lib/kubelet/kubeconfig
done
```

**Result:**
- node-0: `/var/lib/kube-proxy/kubeconfig`, `/var/lib/kubelet/kubeconfig` (uses `node-0.kubeconfig`)
- node-1: `/var/lib/kube-proxy/kubeconfig`, `/var/lib/kubelet/kubeconfig` (uses `node-1.kubeconfig`)

### Server Node

**What you're doing:** Copy control plane component configs to server home directory.

```bash
scp admin.kubeconfig   kube-controller-manager.kubeconfig   kube-scheduler.kubeconfig   root@server:~/
```

**Result:**
- server: `~/admin.kubeconfig`, `~/kube-controller-manager.kubeconfig`, `~/kube-scheduler.kubeconfig`

---

## Step 7: Verify Distribution

### Check worker nodes

```bash
for host in node-0 node-1; do
  echo "=== ${host} ==="
  ssh root@${host} "ls -lh /var/lib/kubelet/kubeconfig /var/lib/kube-proxy/kubeconfig"
done
```

Expected output:

```text
=== node-0 ===
-rw------- 1 root root 9.9K Jan  2 20:26 /var/lib/kube-proxy/kubeconfig
-rw------- 1 root root  10K Jan  2 20:26 /var/lib/kubelet/kubeconfig
=== node-1 ===
-rw------- 1 root root 9.9K Jan  2 20:26 /var/lib/kube-proxy/kubeconfig
-rw------- 1 root root  10K Jan  2 20:26 /var/lib/kubelet/kubeconfig
```

### Check server

```bash
ssh root@server "ls -lh ~/admin.kubeconfig ~/kube-controller-manager.kubeconfig ~/kube-scheduler.kubeconfig"
```

Expected output:

```text
-rw------- 1 root root 9.9K Jan  2 20:27 /root/admin.kubeconfig
-rw------- 1 root root 9.9K Jan  2 20:27 /root/kube-controller-manager.kubeconfig
-rw------- 1 root root 9.9K Jan  2 20:27 /root/kube-scheduler.kubeconfig
```

---

## Understanding `--embed-certs=true`

**What it does:** Embeds certificate content directly in kubeconfig file (base64 encoded) instead of storing file paths.

Without embed (not recommended):

```yaml
clusters:
  - cluster:
      certificate-authority: /path/to/ca.crt  # File path
```

With embed (what you use):

```yaml
clusters:
  - cluster:
      certificate-authority-data: LS0tLS1CRUdJTi...  # Embedded base64 content
```

**Why embed:**
- ✅ Portable: single file contains everything
- ✅ No broken paths when moving files
- ✅ Easier distribution and backup

---

## Kubeconfig Distribution Map

```text
Jumpbox (kubernetes-the-hard-way/)
├── node-0.kubeconfig ────────────> node-0:/var/lib/kubelet/kubeconfig
├── node-1.kubeconfig ────────────> node-1:/var/lib/kubelet/kubeconfig
├── kube-proxy.kubeconfig ────────> Both workers:/var/lib/kube-proxy/kubeconfig
├── kube-controller-manager.kubeconfig ──> server:~/
├── kube-scheduler.kubeconfig ───────────> server:~/
└── admin.kubeconfig ────────────────────> server:~/
```

---

## What Each Component Will Do With Its Kubeconfig

| Component | Kubeconfig Location | Authenticates As | Purpose |
|---|---|---|---|
| kubelet (node-0) | `/var/lib/kubelet/kubeconfig` | `system:node:node-0` | Register node, report status, pull pod assignments |
| kubelet (node-1) | `/var/lib/kubelet/kubeconfig` | `system:node:node-1` | Register node, report status, pull pod assignments |
| kube-proxy | `/var/lib/kube-proxy/kubeconfig` | `system:kube-proxy` | Read service endpoints, configure iptables |
| controller-manager | `~/kube-controller-manager.kubeconfig` | `system:kube-controller-manager` | Watch cluster state, manage controllers |
| scheduler | `~/kube-scheduler.kubeconfig` | `system:kube-scheduler` | Read pod queue, assign pods to nodes |
| kubectl | `~/admin.kubeconfig` | `admin` (`system:masters`) | Full cluster access for administration |

---

## Files Created in Section 5

### On jumpbox (`~/kubernetes-the-hard-way/`)

- `node-0.kubeconfig`
- `node-1.kubeconfig`
- `kube-proxy.kubeconfig`
- `kube-controller-manager.kubeconfig`
- `kube-scheduler.kubeconfig`
- `admin.kubeconfig`

**Total:** 6 kubeconfig files (~10KB each, certificates embedded)

### On VMs

- node-0: 2 files (`/var/lib/kubelet/kubeconfig`, `/var/lib/kube-proxy/kubeconfig`)
- node-1: 2 files (`/var/lib/kubelet/kubeconfig`, `/var/lib/kube-proxy/kubeconfig`)
- server: 3 files (`admin.kubeconfig`, `kube-controller-manager.kubeconfig`, `kube-scheduler.kubeconfig`)
