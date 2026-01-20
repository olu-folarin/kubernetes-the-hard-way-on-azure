# KTHW Section 3: Complete Terminology Breakdown

**Section:** Provisioning Compute Resources

**URL:** https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md

---

## Table of Contents
- [1. Compute Resources](#1-compute-resources)
- [2. Machine Database](#2-machine-database-machinestxt)
- [3. IPv4 Address](#3-ipv4_address)
- [4. FQDN](#4-fqdn-fully-qualified-domain-name)
- [5. Hostname](#5-hostname)
- [6. Pod Subnet](#6-pod_subnet-pod-cidr)
- [7. SSH](#7-ssh-secure-shell)
- [8. Root User](#8-root-user)
- [9. PermitRootLogin](#9-permitrootlogin)
- [10. SSH Key Pair](#10-ssh-key-pair)
- [11. ssh-copy-id](#11-ssh-copy-id)
- [12. while read Loop](#12-while-read-loop)
- [13. /etc/hosts File](#13-etchosts-file)
- [14. hostnamectl](#14-hostnamectl)
- [15. systemd & systemctl](#15-systemd--systemctl)
- [16. FQDN vs IP Trade-offs](#16-fqdn-vs-ip-address-trade-offs)
- [17. DNS Hierarchy](#17-dns-hierarchy)
- [18. sed Command](#18-sed-command)
- [19. scp](#19-scp-secure-copy)
- [20. SSH -n Flag](#20-ssh--n-flag)
- [21. for Loop](#21-for-loop-vs-while-loop)
- [22. Network Topology](#22-network-topology-overview)
- [23. Security Considerations](#23-security-considerations)
- [EKS/AKS Translation](#eksaks-translation)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## **1. COMPUTE RESOURCES**

### What it means:

The physical or virtual machines (VMs) that will run your Kubernetes cluster.

### Role:

These machines form the infrastructure layer - they're the "computers" that Kubernetes software runs on.

### Why:

Kubernetes doesn't exist in the abstract - it needs actual servers/VMs to run its components (API server, etcd, kubelet, etc.). This section provisions those machines.

### Types of compute resources:

- **Control Plane Node** (server) - runs the brain of Kubernetes

- **Worker Nodes** (node-0, node-1) - run your actual applications (containers/pods)

---

## **2. MACHINE DATABASE (`machines.txt`)**

### What it means:

A text file containing critical information about each machine in your cluster.

### Structure:

```

IP_ADDRESS  FQDN  HOSTNAME  POD_SUBNET

```

### Example:

```

10.240.0.10 server.kubernetes.local server

10.240.0.20 node-0.kubernetes.local node-0 10.200.0.0/24

10.240.0.21 node-1.kubernetes.local node-1 10.200.1.0/24

```

### Role:

Acts as a "source of truth" for automation scripts. Instead of hardcoding values, scripts read from this file.

### Why:

- **Single source of truth**: Change one file instead of editing multiple scripts

- **Scriptability**: Loop through machines programmatically

- **Documentation**: Clear record of cluster topology

- **Infrastructure as Code**: Can version control this file

---

## **3. IPV4_ADDRESS**

### What it means:

The IP address assigned to each machine (e.g., `10.240.0.10`)

### Role:

Unique network identifier for each machine on the network.

### Why needed:

- Initial SSH access (before hostnames are configured)

- Network routing (how machines find each other)

- Foundation for all network communication

### My Azure values:

```

10.240.0.10 → server (control plane)

10.240.0.20 → node-0 (worker)

10.240.0.21 → node-1 (worker)

```

---

## **4. FQDN (Fully Qualified Domain Name)**

### What it means:

Complete domain name that specifies exact location in DNS hierarchy.

### Format:

```

[hostname].[domain].[top-level-domain]

server.kubernetes.local

│      │          │

│      │          └─ Top-level domain (local = private network)

│      └─ Domain name

└─ Hostname

```

### Role:

Human-readable name that maps to IP address. More memorable than `10.240.0.10`.

### Why use FQDN instead of just hostname:

- **Uniqueness**: `server` might exist in multiple domains, `server.kubernetes.local` is unique

- **DNS resolution**: Proper DNS hierarchy

- **Convention**: Kubernetes best practice for cluster identification

- **Certificate generation**: TLS certs use FQDNs in Subject Alternative Names (SAN)

### My values:

```

server.kubernetes.local

node-0.kubernetes.local

node-1.kubernetes.local

```

### Why `.local` domain:

- RFC 6762: `.local` is reserved for private networks (mDNS/Bonjour)

- Not routable on public internet (security)

- Indicates "this is internal infrastructure"

---

## **5. HOSTNAME**

### What it means:

Short name for the machine (just `server`, not `server.kubernetes.local`)

### Role:

- System identification (shown in shell prompt: `root@server:~#`)

- Kubernetes node name (how node registers with API server)

- Convenient shorthand for SSH (`ssh root@server` vs `ssh root@server.kubernetes.local`)

### Why both FQDN and hostname:

- **FQDN**: Formal identification, certificates, DNS records

- **Hostname**: Convenience, displayed in commands, logs, shell

### Command to set hostname:

```bash

hostnamectl set-hostname server

```

### How it appears:

```bash

# Before:

root@ubuntu-vm:~#

# After:

root@server:~#

```

---

## **6. POD_SUBNET (Pod CIDR)**

### What it means:

IP address range allocated to each worker node for assigning IPs to pods.

### Format:

```

10.200.0.0/24

│  │   │ │  │

│  │   │ │  └─ /24 = 256 addresses (10.200.0.0 to 10.200.0.255)

│  │   │ └─ Node identifier (0 = node-0, 1 = node-1)

│  │   └─ Pod network prefix

│  └─ Private network range

└─ Private IP space

```

### Role:

Determines IP addresses that pods get when they start on a specific node.

### Why each node gets its own subnet:

- **No IP conflicts**: Node-0 pods get 10.200.0.x, node-1 pods get 10.200.1.x

- **Routing**: Each node knows which IPs belong to which node

- **Scalability**: Can add node-2 with 10.200.2.0/24, node-3 with 10.200.3.0/24, etc.

### Example:

```

node-0 → 10.200.0.0/24

  ├─ nginx pod → 10.200.0.5

  ├─ redis pod → 10.200.0.6

  └─ app pod → 10.200.0.7

node-1 → 10.200.1.0/24

  ├─ database pod → 10.200.1.5

  └─ cache pod → 10.200.1.6

```

### Why control plane (server) has no pod subnet:

- Control plane doesn't run user workloads (no user pods)

- Only runs system components (API server, etcd, scheduler, controller-manager)

- These run as systemd services, not pods (in this tutorial)

---

## **7. SSH (Secure Shell)**

### What it means:

Encrypted network protocol for secure remote access to machines.

### Role:

- Access machines from jumpbox

- Run commands on remote machines

- Copy files between machines (scp)

### Why needed:

- You can't physically access cloud VMs (no keyboard/monitor)

- Security: encrypted connection, prevents password sniffing

- Automation: scripts can SSH to configure machines

### SSH Architecture:

```

Jumpbox                     Target Machine

  ├─ Private key (~/.ssh/id_rsa) ← → Public key (~/.ssh/authorized_keys)

  │

  └─ SSH client (ssh command)  ← → SSH server (sshd daemon)

```

---

## **8. ROOT USER**

### What it means:

The superuser account on Linux with unlimited privileges (UID 0).

### Role:

- Install software (`apt-get install`)

- Modify system files (`/etc/hosts`, `/etc/ssh/sshd_config`)

- Start/stop services (`systemctl restart`)

- Full control over machine

### Why KTHW uses root:

- Simplicity: Don't need `sudo` before every command

- Kubernetes components need root access (bind to low ports, modify networking)

### Security trade-off:

- **Danger**: Typo can destroy system (`rm -rf /` = delete everything)

- **Production**: Use dedicated service accounts, not root

- **KTHW**: Learning environment, root is acceptable

### Alternative (production):

```bash

# Create kubernetes user

adduser kubernetes

usermod -aG sudo kubernetes

# Run commands as kubernetes

sudo systemctl restart kubelet

```

---

## **9. PermitRootLogin**

### What it means:

SSH server configuration that controls if root user can SSH in.

### Why disabled by default:

- **Security**: Root is most valuable target for attackers

- **Brute force**: Attacker knows username (`root`), only needs to guess password

- **Best practice**: SSH as normal user, then `sudo` to root

### File location:

```

/etc/ssh/sshd_config

```

### Setting:

```bash

# Before (secure but inconvenient):

PermitRootLogin no

# After (convenient but less secure):

PermitRootLogin yes

```

### Why KTHW changes it:

- Tutorial uses password-less SSH keys (not passwords)

- Keys are cryptographically strong (4096-bit RSA)

- Convenience for learning environment

### Production alternative:

```bash

PermitRootLogin prohibit-password  # Allow keys, deny passwords

```

---

## **10. SSH KEY PAIR**

### What it means:

Two mathematically related files for authentication without passwords.

### Components:

```

~/.ssh/id_rsa       # Private key (SECRET, stays on jumpbox)

~/.ssh/id_rsa.pub   # Public key (safe to share, goes on target machines)

```

### How it works:

```

1. Jumpbox generates key pair

2. Public key copied to target machine: ~/.ssh/authorized_keys

3. When connecting:

   - Jumpbox: "I want to connect, here's my public key signature"

   - Target: "That signature matches my authorized_keys, welcome!"

```

### Why better than passwords:

- **No password transmission**: Nothing to intercept

- **Strong crypto**: 4096-bit key = effectively unbreakable

- **No typing**: Automated scripts can connect

- **Revocable**: Remove public key = access denied

### Generate keys:

```bash

ssh-keygen

# Creates:

# - /root/.ssh/id_rsa (NEVER share!)

# - /root/.ssh/id_rsa.pub (copy to machines)

```

---

## **11. ssh-copy-id**

### What it means:

Utility that copies your public SSH key to a remote machine.

### Command:

```bash

ssh-copy-id root@10.240.0.10

```

### What it does behind the scenes:

```bash

# 1. Reads your public key:

cat ~/.ssh/id_rsa.pub

# 2. SSH to target machine (asks for password this ONE time)

# 3. Appends public key to:

#    /root/.ssh/authorized_keys

# 4. Future connections: passwordless!

```

### Why the first connection needs password:

- Chicken-and-egg problem: Can't use key auth before key is installed

- After `ssh-copy-id`: Never need password again

### Manual alternative:

```bash

cat ~/.ssh/id_rsa.pub | ssh root@10.240.0.10 'cat >> ~/.ssh/authorized_keys'

```

---

## **12. while read LOOP**

### What it means:

Bash loop that processes each line of a file.

### Code:

```bash

while read IP FQDN HOST SUBNET; do

  ssh-copy-id root@${IP}

done < machines.txt

```

### How it works:

```

1. Read line from machines.txt

2. Split into variables:

   - IP = 10.240.0.10

   - FQDN = server.kubernetes.local

   - HOST = server

   - SUBNET = (empty for control plane)

3. Execute command using variables

4. Repeat for next line

```

### Why this pattern:

- **DRY**: Don't Repeat Yourself (write loop once, not same command 3 times)

- **Scalability**: Add 10 more machines? Loop handles it

- **Consistency**: Same command for all machines

### Equivalent manual commands:

```bash

ssh-copy-id root@10.240.0.10

ssh-copy-id root@10.240.0.20

ssh-copy-id root@10.240.0.21

```

---

## **13. /etc/hosts FILE**

### What it means:

Local DNS lookup table that maps hostnames to IP addresses.

### Format:

```

IP_ADDRESS   FQDN                    HOSTNAME

10.240.0.10  server.kubernetes.local server

10.240.0.20  node-0.kubernetes.local node-0

10.240.0.21  node-1.kubernetes.local node-1

```

### Role:

When you type `ssh root@server`, system checks:

```

1. /etc/hosts first (local override)

2. DNS servers (if not found in hosts file)

```

### Why needed:

- **No DNS server**: This tutorial doesn't set up a DNS server

- **Speed**: Local file faster than DNS query

- **Reliability**: Works even if DNS is down

- **Simplicity**: No external dependencies

### How name resolution works:

```bash

# User types:

ssh root@server

# System does:

1. Look up "server" in /etc/hosts

2. Find: 10.240.0.10

3. Connect to: ssh root@10.240.0.10

```

### Without /etc/hosts:

```bash

ssh root@server

# Error: Could not resolve hostname server: Name or service not known

```

### Production alternative:

- CoreDNS (Kubernetes internal DNS)

- Cloud provider DNS (Route53, Azure DNS)

- Internal DNS server (BIND, dnsmasq)

---

## **14. hostnamectl**

### What it means:

Systemd utility to query and change system hostname.

### Command:

```bash

hostnamectl set-hostname server

```

### What it does:

```

1. Updates /etc/hostname file

2. Updates kernel hostname (uname -n)

3. Updates systemd-hostnamed service

4. Updates shell prompt

```

### Before/after:

```bash

# Before:

root@ip-10-240-0-10:~#

# After:

root@server:~#

```

### Why not just `hostname server`:

- `hostname` command is temporary (resets on reboot)

- `hostnamectl` is persistent (survives reboot)

- `hostnamectl` integrates with systemd properly

### Verify:

```bash

hostnamectl

# Output:

#   Static hostname: server

#   Icon name: computer-vm

#   Chassis: vm

#   Machine ID: abc123...

```

---

## **15. systemd & systemctl**

### What it means:

**systemd** = System and service manager for Linux (PID 1)

**systemctl** = Command to control systemd

### Role:

- Start/stop services (daemons)

- Enable services to start at boot

- View service status/logs

### Common commands:

```bash

systemctl restart sshd           # Restart SSH server

systemctl status kubelet         # Check kubelet status

systemctl enable docker          # Start docker at boot

systemctl daemon-reload          # Reload service definitions

```

### Why restart sshd after config change:

```

1. Edit /etc/ssh/sshd_config (file on disk)

2. SSH daemon still running with old config (in memory)

3. systemctl restart sshd → reload config from disk

```

### Service files location:

```

/etc/systemd/system/          # Custom services

/lib/systemd/system/          # System services

```

---

## **16. FQDN vs IP Address Trade-offs**

### Why not just use IP addresses everywhere?

**Problems with IPs:**

```

1. Hard to remember:

   ssh root@10.240.0.20  # Is this node-0 or node-1?

2. Hard to change:

   - VM gets new IP → update 50 config files

3. Not human-friendly:

   kubectl logs nginx-pod-10.240.0.20  # What node is this?

```

**Benefits of hostnames:**

```

1. Readable:

   ssh root@node-0  # Obvious which machine

2. Flexible:

   - node-0 IP changes → update /etc/hosts once

3. Certificates:

   - TLS certs use hostnames, not IPs

   - CN=server.kubernetes.local

```

---

## **17. DNS HIERARCHY**

### Understanding the structure:

```

server.kubernetes.local

│      │          │

│      │          └─ TLD (Top-Level Domain)

│      └─ Domain

└─ Hostname

Example breakdown:

www.google.com

│   │      │

│   │      └─ .com (TLD)

│   └─ google (Domain)

└─ www (Hostname)

```

### Why `.local`:

- **Private network indicator**: Not routable on internet

- **mDNS/Zeroconf**: Auto-discovery on local networks

- **Security**: Can't accidentally expose internal services

### Production alternatives:

```

server.k8s.company.internal     # Corporate internal domain

server.cluster.prod.aws         # Cloud provider naming

server.eu-west-1.k8s.acme.com   # Geographic + org naming

```

---

## **18. sed COMMAND**

### What it means:

Stream EDitor - edits files programmatically without opening them.

### Command from tutorial:

```bash

sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts

```

### Breakdown:

```

sed -i              # In-place edit (modify file directly)

's/                 # Substitute command

^127.0.1.1.*        # Pattern to find (regex)

/                   # Separator

127.0.1.1\t${FQDN} ${HOST}  # Replacement text

/'                  # End substitute

/etc/hosts          # File to edit

```

### What it does:

```

# Before /etc/hosts:

127.0.1.1   ubuntu-vm

# After:

127.0.1.1   server.kubernetes.local server

```

### Why needed:

Ubuntu VMs have auto-generated hostname tied to 127.0.1.1. This replaces it with my chosen FQDN/hostname.

### Manual alternative:

```bash

vim /etc/hosts

# Find line starting with 127.0.1.1

# Replace with: 127.0.1.1  server.kubernetes.local server

```

---

## **19. scp (Secure Copy)**

### What it means:

Copy files between machines over SSH (encrypted).

### Syntax:

```bash

scp LOCAL_FILE user@remote:/path/to/destination

scp user@remote:/path/to/file LOCAL_DESTINATION

```

### Example from tutorial:

```bash

scp hosts root@server:~/

```

### What it does:

```

1. Read local file: hosts

2. SSH to server as root

3. Copy to remote path: /root/hosts

```

### Why not just `cp`:

- `cp` only works on local filesystem

- `scp` works across network

- Uses SSH for security (encrypted transfer)

### Alternative tools:

```bash

rsync -av file root@server:~/  # More features (resume, delta transfer)

sftp root@server               # Interactive file transfer

```

---

## **20. SSH -n FLAG**

### What it means:

Prevents SSH from reading from stdin (standard input).

### Command:

```bash

ssh -n root@server hostname

```

### Why needed in loops:

```bash

# WITHOUT -n (breaks loop):

while read IP FQDN HOST SUBNET; do

  ssh root@${IP} hostname

done < machines.txt

# Problem: ssh consumes remaining lines from machines.txt!

# WITH -n (works correctly):

while read IP FQDN HOST SUBNET; do

  ssh -n root@${IP} hostname

done < machines.txt

# ssh ignores stdin, loop continues properly

```

### What happens without `-n`:

```

1. Loop reads first line (server)

2. SSH connects and... reads next line from machines.txt!

3. Loop only processes first machine, skips rest

```

---

## **21. for LOOP (vs while LOOP)**

### Difference:

**while loop** (used earlier):

```bash

while read IP FQDN HOST SUBNET; do

  # Process each line from file

done < machines.txt

```

**for loop** (used for verification):

```bash

for host in server node-0 node-1; do

  ssh root@${host} hostname

done

```

### When to use each:

- **while read**: Processing file contents

- **for loop**: Iterating over known list

---

## **22. NETWORK TOPOLOGY OVERVIEW**

### Complete picture after Section 3:

```

Internet

    ↓

Jumpbox (10.240.0.4 private)

    ↓ SSH (kubernetes-vnet: 10.240.0.0/24)

    ├─ server (10.240.0.10)

    │   └─ Runs: kube-apiserver, etcd, kube-scheduler, kube-controller-manager

    │

    ├─ node-0 (10.240.0.20)

    │   ├─ Runs: kubelet, kube-proxy, containerd

    │   └─ Pod CIDR: 10.200.0.0/24

    │       └─ Pods get IPs: 10.200.0.1, 10.200.0.2, etc.

    │

    └─ node-1 (10.240.0.21)

        ├─ Runs: kubelet, kube-proxy, containerd

        └─ Pod CIDR: 10.200.1.0/24

            └─ Pods get IPs: 10.200.1.1, 10.200.1.2, etc.

```

### Network ranges summary:

```

10.240.0.0/16  → Cluster network (VMs)

  └─ 10.240.0.0/24 → kubernetes-subnet (my 4 machines)

10.200.0.0/16  → Pod network (containers)

  ├─ 10.200.0.0/24 → Pods on node-0

  └─ 10.200.1.0/24 → Pods on node-1

```

---

## **23. SECURITY CONSIDERATIONS**

### What I'm doing (KTHW):

- Root SSH access enabled

- Password authentication (initially, then key-based)

- No firewall rules

- VMs in same network

### Production differences:

- **No root SSH**: Dedicated service accounts

- **Key-only authentication**: `PermitRootLogin prohibit-password`

- **Network segmentation**: Control plane in separate subnet

- **Security groups**: Firewall rules (allow only necessary ports)

- **Bastion hardening**: 2FA, session recording, IP whitelisting

- **Certificate-based auth**: Short-lived certs instead of long-lived keys

---

## **SUMMARY: Why This Section Matters**

### What you're building:

```

Foundation for Kubernetes cluster:

✓ Machines can communicate (networking)

✓ Jumpbox can access all machines (SSH keys)

✓ Machines have identities (hostnames, FQDNs)

✓ Name resolution works (/etc/hosts)

✓ Automation ready (machines.txt + loops)

```

### Capabilities after this section:

1. ✅ SSH to any machine by hostname (`ssh root@node-0`)

2. ✅ Machines can find each other (name resolution)

3. ✅ Foundation for Kubernetes networking

4. ✅ Automation-ready infrastructure (scripts can loop through machines)

### Why this foundation is critical:

- **Section 4**: Generate certs for each hostname

- **Section 5**: Create kubeconfig files referencing hostnames

- **Section 7-9**: SSH to each machine to install components

- **Section 11**: Configure routes between machines

### What comes next:

Section 4 will generate TLS certificates for:

- API server authentication

- Kubelet authentication

- etcd cluster security

- Admin access (kubectl)

All these certificates will reference the hostnames I configured in this section.

---

## **KEY TAKEAWAYS**

1. **Infrastructure is code**: `machines.txt` + bash loops = reproducible infrastructure

2. **Names > IPs**: Hostnames provide flexibility, readability, certificate compatibility

3. **SSH keys > passwords**: Cryptographically strong, automation-friendly

4. **Bastion pattern**: Jumpbox isolates cluster from internet (security)

5. **Network design matters**: Separate ranges for VMs and pods prevents conflicts

6. **Automation first**: Setup scales from 3 machines to 300 with same scripts

---

## **EKS/AKS TRANSLATION**

### What EKS/AKS abstract away:

```

✅ VM provisioning (automatic node pools)

✅ Networking setup (VPC/VNet created)

✅ Hostname configuration (handled by cloud provider)

✅ SSH key management (optional, can use SSM/Bastion)

```

### What you still manage:

```

⚠️ Node groups (how many, what size)

⚠️ Pod CIDRs (if using custom networking)

⚠️ Security groups (what traffic is allowed)

⚠️ SSH access (if you need to troubleshoot nodes)

```

### How this knowledge helps with managed K8s:

- **Debugging**: SSH to nodes to check kubelet logs

- **Networking issues**: Understand pod CIDR conflicts

- **Security**: Know what /etc/hosts entries exist

- **Architecture**: Understand what's happening "under the hood"

---

## **TROUBLESHOOTING GUIDE**

### "ssh: Could not resolve hostname server"

**Cause**: /etc/hosts not configured

**Fix**: Check `cat /etc/hosts`, ensure entries exist

### "Permission denied (publickey)"

**Cause**: SSH key not copied to target machine

**Fix**: Run `ssh-copy-id root@IP` again

### "Connection refused"

**Cause**: SSH daemon not running

**Fix**: `systemctl status sshd`, `systemctl start sshd`

### "PermitRootLogin no"

**Cause**: sshd_config blocks root access

**Fix**: Edit `/etc/ssh/sshd_config`, set `PermitRootLogin yes`, restart sshd

### Pods getting wrong IPs

**Cause**: Pod CIDR misconfigured

**Fix**: Check machines.txt, verify each node has unique pod subnet

---

## **ADDITIONAL RESOURCES**

- **SSH**: `man ssh`, `man ssh-keygen`, `man sshd_config`

- **systemd**: `man systemctl`, `man systemd`

- **Networking**: RFC 1918 (private IP ranges), RFC 6762 (.local domain)

- **DNS**: `man hosts`, `/etc/nsswitch.conf` (name resolution order)

- **Bash**: Advanced Bash Scripting Guide (TLDP)