# Section 7: Bootstrapping the etcd Cluster

## Overview

This section bootstraps **etcd** — the distributed key-value database that stores all Kubernetes cluster state and data.

**Library Catalog Analogy:**
- **Kubernetes Cluster** = Library building
- **etcd** = The catalog system (database)
- **API Server** = Librarians (access the catalog)
- **Books** = Cluster resources (pods, services, secrets)
- **Catalog entries** = Records in etcd

Without the catalog (etcd), librarians don't know what books exist, where they are, or who has them. Similarly, without etcd, Kubernetes doesn't know what resources exist or their state.

---

## What is etcd?

**etcd** = Distributed, reliable key-value store used as Kubernetes' backing store for all cluster data.

**What it stores:**
- All cluster resources (pods, services, deployments, secrets, configmaps)
- Cluster state (which pods on which nodes, health status)
- Configuration data (RBAC policies, network policies)
- Watch history (for real-time updates)

**Why it's critical:**
- ❌ If etcd is down → Kubernetes API is down (can't read/write state)
- ❌ If etcd data is corrupted → Cluster loses all state
- ✅ If etcd is healthy → Cluster operates normally

**Key characteristics:**
- **Consistent:** Uses Raft consensus algorithm (all nodes agree on state)
- **Reliable:** Data replicated across multiple nodes (production: 3 or 5)
- **Fast:** Optimized for reads (most Kubernetes operations)
- **Secure:** TLS for all communication

---

## What is "Bootstrapping"?

**Bootstrap** = Initialize and start a service from scratch until it's fully operational.

**Restaurant Opening Analogy:**

| Phase | Restaurant | etcd Bootstrapping |
|------:|-----------|--------------------|
| 1. Preparation | Order equipment, hire staff | Download binaries, create users/dirs |
| 2. Setup | Install kitchen, arrange tables | Copy certificates, configure service |
| 3. Opening | Turn on equipment, unlock doors | Start systemd service |
| 4. Verification | First customer served | Check etcd is responding |
| 5. Operational | Restaurant open for business | etcd ready for API server |

**After bootstrapping:** etcd runs continuously, automatically restarts on failure, and starts on boot.

---

## etcd in Your Architecture

```text
┌──────────────────────────────────────────────────────────┐
│ Server Node (10.240.0.10)                                │
│                                                          │
│  ┌────────────────────────────────────────┐              │
│  │ etcd (port 2379)                        │              │
│  │                                        │              │
│  │ Data Directory: /var/lib/etcd/          │              │
│  │ Config:         /etc/etcd/              │              │
│  │ Certificates:                           │              │
│  │  - ca.crt (verify clients)              │              │
│  │  - kube-apiserver.crt/key (serve)       │              │
│  │                                        │              │
│  │ Listens:                                │              │
│  │  - 127.0.0.1:2379 (clients)             │              │
│  │  - 127.0.0.1:2380 (peers)               │              │
│  └────────────┬───────────────────────────┘              │
│               │                                          │
│  ┌────────────▼───────────────────────────┐              │
│  │ kube-apiserver (Section 8)             │              │
│  │ Will connect to etcd at:               │              │
│  │ https://127.0.0.1:2379                 │              │
│  └────────────────────────────────────────┘              │
└──────────────────────────────────────────────────────────┘
```

**Important:**
- etcd only listens on localhost (`127.0.0.1`) — not exposed to the network
- Only the API server connects directly to etcd
- All other components go through the API server

---

## Tutorial vs Production

| Aspect | Tutorial (Your Setup) | Production |
|--------|------------------------|------------|
| **Instances** | 1 etcd (single node) | 3 or 5 etcd (HA cluster) |
| **Failure tolerance** | None (single point of failure) | 1 node down (3-node) or 2 down (5-node) |
| **Network** | Localhost only | Dedicated network between nodes |
| **Backup** | Manual | Automated snapshots every N hours |
| **Hardware** | Shared with control plane | Dedicated machines (I/O intensive) |
| **Monitoring** | None | Prometheus metrics, alerts |

**Why 3 or 5 in production:** Raft consensus requires a majority (quorum):
- 3 nodes: Tolerates 1 failure (2 still make majority)
- 5 nodes: Tolerates 2 failures (3 still make majority)
- Even numbers are not used (4 = same tolerance as 3, but more overhead)

---

## Step 1: Copy Binaries and Service File to Server

**Purpose:** Transfer etcd binaries and the systemd service file from the jumpbox to the server.

**On jumpbox:**
```bash
scp   downloads/controller/etcd   downloads/client/etcdctl   units/etcd.service   root@server:~/
```

**What you're copying:**
- `etcd` — main etcd server binary
- `etcdctl` — command-line client for etcd (management/troubleshooting)
- `etcd.service` — systemd unit file (tells Linux how to run etcd)

---

## Step 2: Install etcd Binaries

**Purpose:** Move binaries into the system PATH so they're executable from anywhere.

**On server (after SSH'ing to it):**
```bash
ssh root@server
mv etcd etcdctl /usr/local/bin/
```

**Why `/usr/local/bin`:**
- Standard location for locally-installed binaries
- In system PATH (you can run `etcd` from anywhere)
- Survives system updates (unlike `/usr/bin`)

---

## Step 3: Create etcd Directories and Copy Certificates

**Purpose:** Create storage directories and copy TLS certificates for secure communication.

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-apiserver.key kube-apiserver.crt /etc/etcd/
}
```

**What each directory does:**

| Directory | Purpose | Contains |
|----------|---------|----------|
| `/etc/etcd/` | Configuration and certificates | `ca.crt`, `kube-apiserver.crt/key` |
| `/var/lib/etcd/` | Data storage (database files) | etcd database (created automatically) |

**Why `chmod 700`:** Only root can access etcd data (security — contains all cluster secrets).

**Certificates:**
- `ca.crt` — verifies client certificates (API server must have a cert signed by this CA)
- `kube-apiserver.crt/key` — etcd's server certificate (proves identity to clients)

**Note:** You're reusing `kube-apiserver` certificates for etcd (common in single-node setups). Production setups typically use dedicated etcd certificates.

---

## Step 4: Install systemd Service File

**Purpose:** Configure systemd to manage etcd (auto-start, auto-restart).

```bash
mv etcd.service /etc/systemd/system/
```

**What systemd does:**
- Starts etcd on boot
- Restarts etcd if it crashes
- Manages logs (viewable via `journalctl -u etcd`)
- Provides commands: `systemctl start/stop/restart/status etcd`

The `etcd.service` file contains:
- `ExecStart` command (how to run etcd)
- Environment variables
- Restart policies
- Dependencies (what must start before etcd)

---

## Step 5: Start etcd Service

**Purpose:** Enable and start etcd for the first time.

```bash
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

**What each command does:**

| Command | Purpose |
|--------|---------|
| `daemon-reload` | Tell systemd to reload service files (so it sees new `etcd.service`) |
| `enable etcd` | Auto-start etcd on boot (creates symlink) |
| `start etcd` | Start etcd now |

**Expected output (example):**
```text
Created symlink /etc/systemd/system/multi-user.target.wants/etcd.service → /etc/systemd/system/etcd.service.
```

This confirms etcd is enabled and will start on boot.

---

## Step 6: Verify etcd is Running

**Purpose:** Confirm etcd started successfully and is healthy.

### Check member list
```bash
etcdctl member list
```

**Expected output (example):**
```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

What this shows:
- **Member ID:** unique identifier
- **Status:** `started` (etcd is running)
- **Name:** node name (e.g., `controller`)
- **Peer URL:** `http://127.0.0.1:2380` (cluster comms; unused in single-node)
- **Client URL:** `http://127.0.0.1:2379` (API server connects here)
- **Is Learner:** `false` (full member)

### Check health
```bash
etcdctl endpoint health
```

**Expected output (example):**
```text
127.0.0.1:2379 is healthy: successfully committed proposal: took = 1.234ms
```

This confirms etcd can accept and commit writes (fully operational).

---

## Understanding etcd Ports

| Port | Purpose | Who Connects |
|------|---------|--------------|
| `2379` | Client connections | `kube-apiserver` (Section 8) |
| `2380` | Peer connections | Other etcd members (unused in single-node) |

**Why localhost only (`127.0.0.1`):**
- Security: etcd is not exposed to the network
- Only local API server can access it
- Production: would listen on private network IP for HA clusters

---

## etcd Data Structure

How Kubernetes data is stored in etcd:

```text
/registry/
├── pods/
│   ├── default/
│   │   └── my-pod
│   └── kube-system/
│       └── coredns-abc123
├── services/
│   └── default/
│       └── kubernetes
├── secrets/
│   └── default/
│       └── my-secret  (encrypted with encryption-config.yaml!)
├── configmaps/
├── deployments/
└── ...
```

Key format: `/registry/<resource-type>/<namespace>/<name>`

**Example — reading a secret from etcd directly:**
```bash
# After Section 8, when API server is running:
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret   --endpoints=https://127.0.0.1:2379   --cacert=/etc/etcd/ca.crt   --cert=/etc/etcd/kube-apiserver.crt   --key=/etc/etcd/kube-apiserver.key
```

With encryption enabled (Section 6), output starts with:
`k8s:enc:aescbc:v1:key1:` (encrypted data).

---

## Troubleshooting Common Issues

### Issue: etcd won't start
Check logs:
```bash
journalctl -u etcd -f
```

Common causes:
- Port 2379 already in use
- Certificate permissions wrong
- Data directory permissions wrong
- Certificate path incorrect in `etcd.service`

Fix permissions:
```bash
chmod 700 /var/lib/etcd
chmod 644 /etc/etcd/*.crt
chmod 600 /etc/etcd/*.key
```

### Issue: "connection refused" when using etcdctl
Check if etcd is running:
```bash
systemctl status etcd
```

Check if it's listening on 2379:
```bash
ss -tlnp | grep 2379
```

Expected: etcd listening on `127.0.0.1:2379`.

---

## Useful etcd Commands

Check cluster health:
```bash
etcdctl endpoint health
```

List all keys (after cluster is populated):
```bash
ETCDCTL_API=3 etcdctl get / --prefix --keys-only
```

Get etcd version:
```bash
etcd --version
etcdctl version
```

Check etcd service status:
```bash
systemctl status etcd
```

View etcd logs:
```bash
journalctl -u etcd -f     # Follow logs (live)
journalctl -u etcd -n 50  # Last 50 lines
```

Restart etcd:
```bash
systemctl restart etcd
```

---

## Files Created/Modified in Section 7

**On server:**

**Binaries:**
- `/usr/local/bin/etcd` — etcd server
- `/usr/local/bin/etcdctl` — etcd client

**Configuration:**
- `/etc/systemd/system/etcd.service` — systemd unit file
- `/etc/etcd/ca.crt` — CA certificate
- `/etc/etcd/kube-apiserver.crt` — server certificate
- `/etc/etcd/kube-apiserver.key` — server private key

**Data:**
- `/var/lib/etcd/` — etcd database (auto-created on first start)

**systemd:**
- `/etc/systemd/system/multi-user.target.wants/etcd.service` — symlink (auto-start on boot)

---