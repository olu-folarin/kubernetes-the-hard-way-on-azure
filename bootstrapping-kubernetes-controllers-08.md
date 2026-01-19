# Section 8: Bootstrapping the Kubernetes Control Plane

## Overview

This section bootstraps the Kubernetes control plane on the server node - the "brain" of the cluster that manages all cluster operations.

**Office Building Analogy:**
- **kube-apiserver** = Front desk receptionist (handles all requests, validates credentials, routes to right department)
- **etcd** = Filing room (stores all company records)
- **kube-controller-manager** = Department supervisors (ensure each department maintains desired state)
- **kube-scheduler** = Assignment coordinator (decides which employee does which task)
- **Workers** = Employees (do the actual work)

Without the control plane, the cluster has no coordination - workers wouldn't know what to do or where to send results.

---

## The Control Plane Components

### 1. kube-apiserver

**What:** REST API server - the single entry point for ALL cluster operations.

**Purpose:**
- Validates and processes API requests (from kubectl, kubelets, controllers)
- Reads/writes cluster state to etcd
- Authenticates users via certificates
- Authorizes requests via RBAC
- Applies admission controllers (validation, mutation)

**Who talks to it:**
- kubectl (admin commands)
- kubelets (worker nodes reporting status)
- kube-scheduler (watching for unscheduled pods)
- kube-controller-manager (watching cluster state)
- External tools (monitoring, CI/CD)

**Analogy:** Front desk at a company - everyone must go through it, nobody talks directly to the filing room (etcd).

---

### 2. kube-controller-manager

**What:** Runs controller processes that watch cluster state and take corrective action.

**Purpose:**
- Ensures desired state matches actual state
- Runs built-in controllers (deployment, replicaset, node, service, etc.)
- Each controller watches specific resources and reacts to changes

**Example controllers:**
- **Node Controller:** Monitors nodes, marks unhealthy nodes
- **Replication Controller:** Ensures correct number of pod replicas
- **Endpoints Controller:** Populates endpoints for services
- **Service Account Controller:** Creates default service accounts for new namespaces

**Analogy:** Building supervisors who constantly check if everything is as it should be and fix problems (e.g., "We need 3 copies of this pod, but only 2 exist - create 1 more").

---

### 3. kube-scheduler

**What:** Watches for newly created pods with no assigned node and assigns them to nodes.

**Purpose:**
- Decides which worker node should run each pod
- Considers resource requirements (CPU, memory)
- Considers constraints (node affinity, taints/tolerations)
- Balances workload across nodes

**How it works:**
1. Watches API server for unscheduled pods
2. Filters nodes (which can run this pod?)
3. Scores nodes (which is best?)
4. Assigns pod to highest-scoring node
5. API server updates pod with node assignment
6. Kubelet on that node sees assignment and starts pod

**Analogy:** Assignment coordinator who looks at task requirements and employee capabilities, then assigns tasks to the best-suited employee.

---

## Control Plane Architecture

```text
┌────────────────────────────────────────────────────────────────┐
│ Server Node (10.240.0.10)                                       │
│                                                                │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ etcd :2379                                                │   │
│ │ (Database - Cluster State)                                 │   │
│ └────────────────────────┬─────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ kube-apiserver :6443                                      │   │
│ │ (REST API - Single Entry Point)                            │   │
│ │                                                           │   │
│ │ Features:                                                  │   │
│ │ • Authentication (TLS certificates)                        │   │
│ │ • Authorization (RBAC)                                     │   │
│ │ • Admission Control (validation/mutation)                  │   │
│ │ • etcd interface (read/write cluster state)                │   │
│ │ • Secret encryption (using encryption-config.yaml)         │   │
│ └─────────────┬──────────────────────────┬─────────────────┘   │
│               │                          │                      │
│ ┌───────────▼────────────┐      ┌────────▼──────────────┐       │
│ │ kube-controller-manager│      │ kube-scheduler         │       │
│ │                        │      │                       │       │
│ │ Watches cluster state  │      │ Assigns pods to nodes │       │
│ │ Maintains desired state│      │ Watches unscheduled   │       │
│ └────────────────────────┘      └───────────────────────┘       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
             │                                  │
             ▼                                  ▼
          kubectl                            Worker Nodes
       (from jumpbox)                   (kubelets report status)
```

---

## Step 1: Copy Binaries to Server

**Purpose:** Transfer Kubernetes control plane binaries from jumpbox to server.

**On jumpbox:**
```bash
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  root@server:~/
```

**What you're copying:**
- `kube-apiserver` - API server binary
- `kube-controller-manager` - Controller manager binary
- `kube-scheduler` - Scheduler binary
- `kubectl` - Kubernetes command-line client (for verification)

---

## Step 2: Install Binaries

**Purpose:** Move binaries to system PATH so they're executable from anywhere.

**On server (after SSH'ing to it):**
```bash
ssh root@server

{
  mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
}
```

**Why `/usr/local/bin`:**
- Standard location for locally-installed binaries
- In system PATH (can run commands from anywhere)
- Survives system updates

---

## Step 3: Configure kube-apiserver

### Move Certificates and Configs

**Purpose:** Organize certificates and encryption config in a standard location.

```bash
{
  mkdir -p /var/lib/kubernetes/

  mv ca.crt ca.key \
    kube-apiserver.key kube-apiserver.crt \
    service-accounts.key service-accounts.crt \
    encryption-config.yaml \
    /var/lib/kubernetes/
}
```

**What each file does:**

| File | Purpose |
|---|---|
| `ca.crt` | Verifies client certificates |
| `ca.key` | Signs service account tokens |
| `kube-apiserver.crt/key` | API server's TLS certificate |
| `service-accounts.crt/key` | Signs/verifies JWT tokens for pods |
| `encryption-config.yaml` | Secret encryption configuration |

### Install systemd Service

**Purpose:** Configure systemd to manage kube-apiserver (auto-start, auto-restart).

```bash
mv kube-apiserver.service /etc/systemd/system/kube-apiserver.service
```

**Key flags in `kube-apiserver.service`:**
- `--etcd-servers=http://127.0.0.1:2379` - Connect to etcd
- `--encryption-provider-config` - Enable secret encryption
- `--service-cluster-ip-range` - IP range for services
- `--authorization-mode=Node,RBAC` - Enable RBAC
- `--client-ca-file` - Verify client certificates
- `--tls-cert-file`, `--tls-private-key-file` - Server TLS certificate

---

## Step 4: Configure kube-controller-manager

**Purpose:** Set up controller manager to watch and maintain cluster state.

```bash
# Move kubeconfig (tells controller-manager how to connect to API)
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

# Install systemd service
mv kube-controller-manager.service /etc/systemd/system/
```

**Key flags in `kube-controller-manager.service`:**
- `--cluster-cidr=10.200.0.0/16` - Pod IP range
- `--service-cluster-ip-range` - Service IP range
- `--kubeconfig` - How to connect to API server
- `--use-service-account-credentials` - Use individual service accounts for each controller

---

## Step 5: Configure kube-scheduler

**Purpose:** Set up scheduler to assign pods to nodes.

```bash
# Move kubeconfig
mv kube-scheduler.kubeconfig /var/lib/kubernetes/

# Move scheduler config
mv kube-scheduler.yaml /etc/kubernetes/config/

# Install systemd service
mv kube-scheduler.service /etc/systemd/system/
```

**What `kube-scheduler.yaml` contains:**
- API version and kind (`KubeSchedulerConfiguration`)
- Client connection settings (how to talk to API server)
- Leader election settings (for HA setups)

---

## Step 6: Start Control Plane Services

**Purpose:** Enable and start all three control plane components.

```bash
{
  systemctl daemon-reload

  systemctl enable kube-apiserver \
    kube-controller-manager kube-scheduler

  systemctl start kube-apiserver \
    kube-controller-manager kube-scheduler
}
```

**What each command does:**

| Command | Purpose |
|---|---|
| `daemon-reload` | Tell systemd to reload service files |
| `enable` | Auto-start services on boot |
| `start` | Start services now |

**Expected output (example):**
```text
Created symlink /etc/systemd/system/multi-user.target.wants/kube-apiserver.service → /etc/systemd/system/kube-apiserver.service.
Created symlink /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service → /etc/systemd/system/kube-controller-manager.service.
Created symlink /etc/systemd/system/multi-user.target.wants/kube-scheduler.service → /etc/systemd/system/kube-scheduler.service.
```

---

## Step 7: Verify Services are Running

Check if services are active:

```bash
systemctl is-active kube-apiserver
systemctl is-active kube-controller-manager
systemctl is-active kube-scheduler
```

**Expected:** All should return `active`.

Detailed status check:

```bash
systemctl status kube-apiserver
```

Check logs if needed:

```bash
journalctl -u kube-apiserver -n 50
```

---

## Troubleshooting: File Naming Issue

**Issue:** kube-apiserver stuck in `activating` state.

**Error in logs:**
```text
err="open /var/lib/kubernetes/kube-api-server.crt: no such file or directory"
```

**Cause:** Service file references `kube-api-server.crt` (with hyphens), but actual file is `kube-apiserver.crt` (no hyphens).

**Fix:**
```bash
sed -i 's/kube-api-server/kube-apiserver/g' /etc/systemd/system/kube-apiserver.service
systemctl daemon-reload
systemctl restart kube-apiserver
```

Verify:
```bash
systemctl is-active kube-apiserver
# Should return: active
```

---

## Step 8: Verify Control Plane from Server

Check cluster info:

```bash
kubectl cluster-info --kubeconfig admin.kubeconfig
```

Expected output:

```text
Kubernetes control plane is running at https://127.0.0.1:6443
```

---

## Step 9: Configure RBAC for Kubelet Authorization

**Purpose:** Grant API server permission to access kubelet API on worker nodes.

**Why needed:**
- API server needs to retrieve metrics from kubelets
- API server needs to fetch logs from pods
- API server needs to execute commands in pods (`kubectl exec`)

Create ClusterRole and ClusterRoleBinding:

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig
```

Expected output:

```text
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created
```

**What this grants:**
- Permission to get/list/watch nodes
- Permission to get pod logs
- Permission to get pod metrics
- Permission to execute commands in pods

---

## Step 10: Verify from Jumpbox

**Purpose:** Verify control plane is accessible externally.

Exit server and return to jumpbox:

```bash
exit
```

From jumpbox:

```bash
curl --cacert ca.crt https://server.kubernetes.local:6443/version
```

Expected output (example):

```json
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```

This confirms the API server is:
- ✅ Running and responding
- ✅ TLS configured correctly (verified via `ca.crt`)
- ✅ Accessible from jumpbox

---

## How the Control Plane Works Together

### Example: Creating a Deployment

1. `kubectl` (jumpbox) → `kube-apiserver`  
   "Create deployment: nginx, 3 replicas"

2. `kube-apiserver` validates request:
   - Authenticates: Checks `admin.crt` is signed by CA
   - Authorizes: Checks `admin` (`system:masters`) can create deployments
   - Validates: Deployment spec is valid

3. `kube-apiserver` writes to etcd:
   - Stores deployment object
   - Returns success to `kubectl`

4. `kube-controller-manager` (watching deployments):
   - Sees new deployment
   - Creates ReplicaSet (3 replicas)
   - Writes ReplicaSet to API server → etcd

5. `kube-controller-manager` (watching ReplicaSets):
   - Sees new ReplicaSet needs 3 pods
   - Creates 3 pod objects
   - Writes pods to API server → etcd (pods are "unscheduled")

6. `kube-scheduler` (watching unscheduled pods):
   - Sees 3 unscheduled pods
   - Evaluates nodes (`node-0`, `node-1`)
   - Assigns pods to nodes based on resources
   - Updates `pod.spec.nodeName` via API server → etcd

7. `kubelet` on `node-0` (watching assigned pods):
   - Sees 2 pods assigned to it
   - Pulls nginx container image
   - Starts containers
   - Reports status back to API server → etcd

8. `kubelet` on `node-1`:
   - Sees 1 pod assigned to it
   - Starts container
   - Reports status

9. `kube-controller-manager` (watching ReplicaSet):
   - Sees 3/3 pods running
   - Deployment rollout complete ✅

---

## Component Communication Flow

```text
┌─────────────┐
│   kubectl   │ (admin from jumpbox)
└──────┬──────┘
       │ TLS (admin.crt)
       ▼
┌──────────────────────────────────────────┐
│         kube-apiserver :6443             │
│  • Authenticates via certificates        │
│  • Authorizes via RBAC                   │
│  • Validates requests                    │
└─────┬──────────┬─────────────┬───────────┘
      │          │             │
      │          │             │ All watch API server
      │          │             │ for changes
      ▼          ▼             ▼
┌──────────┐ ┌─────────┐ ┌──────────────┐
│   etcd   │ │ kube-   │ │ kube-        │
│          │ │ ctrl-   │ │ scheduler    │
│ (storage)│ │ manager │ │              │
└──────────┘ └─────────┘ └──────────────┘
                 │              │
                 │              │
                 ▼              ▼
           Takes action    Assigns pods
           to maintain     to nodes
           desired state
```

---

## Files Created/Modified in Section 8

### Binaries
- `/usr/local/bin/kube-apiserver`
- `/usr/local/bin/kube-controller-manager`
- `/usr/local/bin/kube-scheduler`
- `/usr/local/bin/kubectl`

### Configuration
- `/var/lib/kubernetes/ca.crt`
- `/var/lib/kubernetes/ca.key`
- `/var/lib/kubernetes/kube-apiserver.crt`
- `/var/lib/kubernetes/kube-apiserver.key`
- `/var/lib/kubernetes/service-accounts.crt`
- `/var/lib/kubernetes/service-accounts.key`
- `/var/lib/kubernetes/encryption-config.yaml`
- `/var/lib/kubernetes/kube-controller-manager.kubeconfig`
- `/var/lib/kubernetes/kube-scheduler.kubeconfig`
- `/etc/kubernetes/config/kube-scheduler.yaml`

### Systemd Services
- `/etc/systemd/system/kube-apiserver.service`
- `/etc/systemd/system/kube-controller-manager.service`
- `/etc/systemd/system/kube-scheduler.service`

### RBAC
- ClusterRole: `system:kube-apiserver-to-kubelet`
- ClusterRoleBinding: `system:kube-apiserver`
