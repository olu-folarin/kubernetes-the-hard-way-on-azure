# KTHW on Azure - Section 11: Provisioning Pod Network Routes

**KTHW Tutorial:** https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/11-pod-network-routes.md

---

## **Section Overview**

**Goal:** Configure network routes so pods on different worker nodes can communicate directly with each other.

**The Problem:**
```
node-0 pods: 10.200.0.0/24
node-1 pods: 10.200.1.0/24

Without routes:
  Pod 10.200.0.5 → 10.200.1.10 = ❌ No route, packet dropped

With routes:
  Pod 10.200.0.5 → 10.200.1.10 = ✅ Routed via node-1
```

**Capabilities After This Section:**
1. ✅ Pods on node-0 can reach pods on node-1
2. ✅ Pods on node-1 can reach pods on node-0
3. ✅ Server can reach pods on both workers
4. ✅ Cross-node pod communication enabled

---

## **Why This Section Matters**

### **The Networking Challenge**

Each worker node has its own pod subnet:
- **node-0**: Pods get IPs from `10.200.0.0/24`
- **node-1**: Pods get IPs from `10.200.1.0/24`

**The Problem:**
When a pod on node-0 (10.200.0.5) tries to reach a pod on node-1 (10.200.1.10), the packet has nowhere to go:

```
Pod 10.200.0.5 on node-0
    │
    ▼
node-0 routing table: "10.200.1.0/24? Never heard of it"
    │
    ▼
❌ Packet dropped
```

### **The Solution**

Add static routes telling each node where to send traffic for remote pod subnets:

```
Pod 10.200.0.5 on node-0
    │
    ▼
node-0 routing table: "10.200.1.0/24 → send to 10.240.0.21 (node-1)"
    │
    ▼
node-1 receives packet, delivers to pod 10.200.1.10
    │
    ▼
✅ Success
```

---

## **Network Topology**

### **Before Routes:**
```
┌─────────────────┐          ┌─────────────────┐
│     node-0      │          │     node-1      │
│   10.240.0.20   │          │   10.240.0.21   │
│                 │    ❌    │                 │
│ ┌─────────────┐ │  No route│ ┌─────────────┐ │
│ │ Pod Network │ │◄────────►│ │ Pod Network │ │
│ │10.200.0.0/24│ │          │ │10.200.1.0/24│ │
│ └─────────────┘ │          │ └─────────────┘ │
└─────────────────┘          └─────────────────┘
```

### **After Routes:**
```
┌─────────────────┐          ┌─────────────────┐
│     node-0      │          │     node-1      │
│   10.240.0.20   │          │   10.240.0.21   │
│                 │    ✅    │                 │
│ ┌─────────────┐ │  Routed  │ ┌─────────────┐ │
│ │ Pod Network │ │◄────────►│ │ Pod Network │ │
│ │10.200.0.0/24│ │          │ │10.200.1.0/24│ │
│ └─────────────┘ │          │ └─────────────┘ │
└─────────────────┘          └─────────────────┘
        │                            │
        └──────────┬─────────────────┘
                   ▼
           ┌─────────────┐
           │   server    │
           │ 10.240.0.10 │
           │  (has routes│
           │  to both)   │
           └─────────────┘
```

---

## **PART 1: Extract Network Information**

### **Step 1.1: Parse machines.txt**

**Command:**
```bash
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}
```

**What It Does:**
Reads machines.txt and extracts IPs and subnets into shell variables.

**Analogy:**
Looking up addresses in your contact book before writing letters.

**How It Works:**

**machines.txt content:**
```
10.240.0.10 server.kubernetes.local server
10.240.0.20 node-0.kubernetes.local node-0 10.200.0.0/24
10.240.0.21 node-1.kubernetes.local node-1 10.200.1.0/24
```

**Parsing:**
- `grep node-0` → Finds the line containing "node-0"
- `cut -d " " -f 1` → Splits by space, takes field 1 (IP address)
- `cut -d " " -f 4` → Splits by space, takes field 4 (pod subnet)

**Result:**
```bash
SERVER_IP=10.240.0.10
NODE_0_IP=10.240.0.20
NODE_0_SUBNET=10.200.0.0/24
NODE_1_IP=10.240.0.21
NODE_1_SUBNET=10.200.1.0/24
```

**Why Variables:**
- Avoids hardcoding IPs in commands
- Single source of truth (machines.txt)
- Easy to adapt if IPs change

---

## **PART 2: Add Routes to Server**

### **Step 2.1: Configure Server Routes**

**Command:**
```bash
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

**What It Does:**
Tells the server that:
- Traffic for node-0's pods (10.200.0.0/24) goes to 10.240.0.20
- Traffic for node-1's pods (10.200.1.0/24) goes to 10.240.0.21

**Analogy:**
Like a postal service learning "All mail for ZIP code 10200 goes through distribution center A, ZIP code 10201 through center B."

**Technical Details:**
Adds kernel routing table entries directing pod CIDR traffic to the appropriate worker node gateway.

**Commands Executed:**
```bash
ip route add 10.200.0.0/24 via 10.240.0.20
ip route add 10.200.1.0/24 via 10.240.0.21
```

**Result on Server:**
```
10.200.0.0/24 via 10.240.0.20 dev eth0  # node-0's pods
10.200.1.0/24 via 10.240.0.21 dev eth0  # node-1's pods
```

**Why Server Needs Routes:**
- API server may need to reach pods (exec, logs, port-forward)
- Controller manager monitors pod health
- Any control plane → pod communication

---

## **PART 3: Add Routes to Worker Nodes**

### **Step 3.1: Configure Node-0 Routes**

**Command:**
```bash
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

**What It Does:**
Tells node-0 that traffic for node-1's pod subnet should be sent to node-1.

**Analogy:**
Node-0 learns "If someone asks for an address in the 10.200.1.x neighborhood, send them to node-1 at 10.240.0.21."

**Technical Details:**
Adds route for remote pod CIDR (10.200.1.0/24) via peer worker node.

**Command Executed:**
```bash
ip route add 10.200.1.0/24 via 10.240.0.21
```

**Result on Node-0:**
```
10.200.1.0/24 via 10.240.0.21 dev eth0
```

**Note:** Node-0 doesn't need a route to 10.200.0.0/24 - that's its own pod network, already reachable locally.

---

### **Step 3.2: Configure Node-1 Routes**

**Command:**
```bash
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
```

**What It Does:**
Tells node-1 that traffic for node-0's pod subnet should be sent to node-0.

**Analogy:**
Node-1 learns "If someone asks for an address in the 10.200.0.x neighborhood, send them to node-0 at 10.240.0.20."

**Technical Details:**
Adds route for remote pod CIDR (10.200.0.0/24) via peer worker node.

**Command Executed:**
```bash
ip route add 10.200.0.0/24 via 10.240.0.20
```

**Result on Node-1:**
```
10.200.0.0/24 via 10.240.0.20 dev eth0
```

---

## **PART 4: Verification**

### **Step 4.1: Verify Server Routes**

**Command:**
```bash
ssh root@server ip route
```

**Expected Output (relevant lines):**
```
10.200.0.0/24 via 10.240.0.20 dev eth0
10.200.1.0/24 via 10.240.0.21 dev eth0
```

**What This Shows:**
Server knows how to reach pods on both workers.

---

### **Step 4.2: Verify Node-0 Routes**

**Command:**
```bash
ssh root@node-0 ip route
```

**Expected Output (relevant lines):**
```
10.200.0.0/24 dev cnio0 proto kernel scope link src 10.200.0.1
10.200.1.0/24 via 10.240.0.21 dev eth0
```

**What This Shows:**
- `10.200.0.0/24 dev cnio0` → Local pod network (via CNI bridge)
- `10.200.1.0/24 via 10.240.0.21` → Remote pod network (via node-1)

---

### **Step 4.3: Verify Node-1 Routes**

**Command:**
```bash
ssh root@node-1 ip route
```

**Expected Output (relevant lines):**
```
10.200.0.0/24 via 10.240.0.20 dev eth0
10.200.1.0/24 dev cnio0 proto kernel scope link src 10.200.1.1
```

**What This Shows:**
- `10.200.0.0/24 via 10.240.0.20` → Remote pod network (via node-0)
- `10.200.1.0/24 dev cnio0` → Local pod network (via CNI bridge)

---

## **How Pod-to-Pod Communication Works**

### **Same Node (No Routes Needed)**
```
Pod A (10.200.0.5) → Pod B (10.200.0.10)
    │
    ▼
CNI bridge (cnio0) handles local delivery
    │
    ▼
Pod B receives packet
✅ Local routing only
```

### **Cross-Node (Routes Required)**
```
Pod A on node-0 (10.200.0.5) → Pod B on node-1 (10.200.1.10)
    │
    ▼
node-0 kernel: "10.200.1.0/24 → via 10.240.0.21"
    │
    ▼
Packet sent to node-1 (10.240.0.21) over VNet
    │
    ▼
node-1 kernel: "10.200.1.10 is local" (cnio0 bridge)
    │
    ▼
CNI bridge delivers to Pod B
    │
    ▼
✅ Cross-node communication successful
```

---

## **Route Summary**

### **Server Routing Table (Additions)**
| Destination | Gateway | Meaning |
|-------------|---------|---------|
| 10.200.0.0/24 | 10.240.0.20 | node-0's pods → send to node-0 |
| 10.200.1.0/24 | 10.240.0.21 | node-1's pods → send to node-1 |

### **Node-0 Routing Table (Additions)**
| Destination | Gateway | Meaning |
|-------------|---------|---------|
| 10.200.0.0/24 | cnio0 (local) | Own pods → local bridge |
| 10.200.1.0/24 | 10.240.0.21 | node-1's pods → send to node-1 |

### **Node-1 Routing Table (Additions)**
| Destination | Gateway | Meaning |
|-------------|---------|---------|
| 10.200.0.0/24 | 10.240.0.20 | node-0's pods → send to node-0 |
| 10.200.1.0/24 | cnio0 (local) | Own pods → local bridge |

---

## **Production Differences**

### **KTHW (This Tutorial):**
- Manual static routes on each node
- Works only within single VNet/VPC
- 2 nodes = 2 routes to manage
- Routes don't survive reboot (non-persistent)

### **Production Environments:**

| Aspect | KTHW | Production |
|--------|------|------------|
| **Route Management** | Manual `ip route` | CNI plugin (automatic) |
| **Scalability** | 2 nodes | 100+ nodes |
| **Persistence** | Lost on reboot | Persistent/dynamic |
| **Cross-VPC** | Not supported | Overlay networks (VXLAN) |
| **Security** | Open | Network policies |

### **Production CNI Plugins:**

**Calico:**
- Uses BGP for route distribution
- Supports network policies
- No overlay (pure L3 routing)

**Cilium:**
- eBPF-based networking
- Advanced network policies
- L7 visibility

**Flannel:**
- Simple overlay network
- VXLAN encapsulation
- Easy to set up

**AWS VPC CNI:**
- Native VPC networking
- Pods get real VPC IPs
- No overlay needed

### **What CNI Handles Automatically:**
```
✅ Route distribution to all nodes
✅ Route updates when nodes join/leave
✅ Persistence across reboots
✅ Cross-subnet/cross-VPC routing
✅ Network policy enforcement
```

---

## **Azure-Specific Considerations**

### **Why This Works in Azure VNet:**

Azure VNets allow direct routing between VMs in the same subnet. My routes work because:
1. All nodes in same VNet (kubernetes-vnet)
2. All nodes in same subnet (kubernetes-subnet)
3. No Azure Network Security Group blocking traffic
4. IP forwarding enabled (Azure allows pod IPs)

### **Production Azure Alternative:**

For production AKS clusters, Azure uses:
- **Azure CNI**: Pods get IPs from VNet subnet directly
- **Kubenet**: Similar to KTHW but with Azure route tables
- **Azure Route Tables**: Managed routing, not manual `ip route`

---

## **Troubleshooting**

### **Issue: "Network is unreachable"**

**Cause:** Route not added or wrong gateway IP

**Check:**
```bash
ssh root@node-0 ip route | grep 10.200.1
```

**Fix:** Re-run the `ip route add` command

---

### **Issue: Pods can't reach pods on other nodes**

**Cause:** Routes missing or IP forwarding disabled

**Check:**
```bash
# Verify routes exist
ssh root@node-0 ip route

# Check IP forwarding
ssh root@node-0 cat /proc/sys/net/ipv4/ip_forward
# Should be 1
```

**Fix:** Add missing routes, enable IP forwarding if needed

---

### **Issue: Routes disappear after reboot**

**Cause:** `ip route add` is not persistent

**Fix (for learning):** Re-run route commands after reboot

**Fix (persistent):** Add routes to `/etc/netplan/*.yaml` or use a CNI plugin

---

### **Issue: "RTNETLINK answers: File exists"**

**Cause:** Route already exists

**Check:**
```bash
ssh root@node-0 ip route | grep 10.200
```

**Fix:** Route is already there, no action needed

---

## **Section 11 Accomplishments**

### **Routes Configured:**
- ✅ Server has routes to both pod subnets
- ✅ Node-0 has route to node-1's pod subnet
- ✅ Node-1 has route to node-0's pod subnet

### **Connectivity Enabled:**
- ✅ Cross-node pod communication possible
- ✅ Control plane can reach pods on any worker
- ✅ Foundation for Kubernetes services/deployments

### **Verification:**
- ✅ `ip route` shows correct entries on all nodes
- ✅ Ready for pod networking tests in Section 12

---

## **Key Takeaways**

1. **Pod subnets are per-node** - Each worker has its own CIDR, routes connect them

2. **Routes are the glue** - Without routes, pod networks are isolated islands

3. **CNI automates this** - Production clusters use CNI plugins instead of manual routes

4. **Kernel routing table** - `ip route` modifies Linux kernel's routing decisions

5. **Not persistent** - These routes are lost on reboot (CNI plugins handle persistence)

6. **Same VNet required** - Direct routing only works within same network; cross-network needs overlays

---

## **Next: Section 12 - Smoke Test**

Now that networking is configured, I can:
1. Deploy test pods and verify they run
2. Test cross-node pod communication
3. Verify services, DNS, and secrets work
4. Confirm the cluster is fully functional

---

## **Commands Reference**

```bash
# Extract variables from machines.txt
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}

# Add routes on server
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF

# Add route on node-0
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF

# Add route on node-1
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF

# Verify routes
ssh root@server ip route
ssh root@node-0 ip route
ssh root@node-1 ip route
```
