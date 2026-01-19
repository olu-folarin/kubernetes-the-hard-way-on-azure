# KTHW on Azure - Section 3: Provisioning Compute Resources

**KTHW Tutorial:** https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md

---

## **Section Overview**

**What I Built:**

```

Jumpbox (10.240.0.4)

  ↓ SSH access via hostnames

  ├─ server (10.240.0.10) - Control plane

  ├─ node-0 (10.240.0.20) - Worker, Pod CIDR: 10.200.0.0/24

  └─ node-1 (10.240.0.21) - Worker, Pod CIDR: 10.200.1.0/24

```

**Capabilities Achieved:**

1. ✅ SSH to any machine by hostname (`ssh root@server`)

2. ✅ Machines know their own identities (hostnames set)

3. ✅ All machines can resolve each other's names

4. ✅ Full mesh connectivity verified

5. ✅ Foundation ready for K8s component installation

---

## **Why This Section Matters**

### **The Foundation Layer**

Kubernetes is just software - it needs actual computers (VMs) to run on. This section provisions those machines and configures them to work together.

**Three Critical Requirements:**

1. **Networking** - Machines must be able to communicate

2. **Identity** - Each machine needs a unique, stable name

3. **Access** - I need secure remote access to configure them

Without this foundation, the remaining KTHW sections cannot proceed.

---

## **Challenges I Faced (Azure-Specific)**

### **Challenge 1: Azure Quota Limits**

**Problem:** New Azure free trial accounts have a 4 vCPU limit per region.

**Impact:**

- Needed: 8 vCPUs (jumpbox: 2, server: 2, node-0: 2, node-1: 2)

- Had: 4 vCPUs limit

**Solution:** Upgraded to pay-as-you-go plan (increased limit to 10 vCPUs)

**Lesson:** Always check quotas before provisioning cloud resources.

---

### **Challenge 2: Network Isolation**

**Problem:** Jumpbox was created in a different VNet than cluster VMs.

**What Happened:**

```

Jumpbox VNet: 10.0.0.0/16 (auto-created by Azure)

    Jumpbox: 10.0.0.4

Cluster VNet: 10.240.0.0/16 (kubernetes-vnet)

    server:  10.240.0.10

    node-0:  10.240.0.20

    node-1:  10.240.0.21

Result: Jumpbox couldn't reach cluster VMs (connection timeout)

```

**Solution:** Deleted and recreated jumpbox in kubernetes-vnet (10.240.0.4)

**Lesson:** All VMs that need to communicate must be in the same VNet (or use VNet peering).

---

### **Challenge 3: Azure Root User Restrictions**

**Problem:** Azure Ubuntu images don't allow `root` as initial username.

**Error:**

```

This user name 'root' meets the general requirements, but is specifically disallowed for this image.

```

**Solution:**

1. Created VMs with `ubuntu` user

2. After creation, enabled root SSH

3. Copied SSH keys to root user

**Lesson:** Cloud providers add security restrictions; need to work within their constraints.

---

## **PART 1: Provision Azure Infrastructure**

### **Step 1.1: Create Kubernetes VNet**

**Command:**

```bash

az network vnet create \

  --name kubernetes-vnet \

  --address-prefix 10.240.0.0/16 \

  --subnet-name kubernetes-subnet \

  --subnet-prefix 10.240.0.0/24

```

**What It Does:**

- Creates isolated virtual network for cluster

- `10.240.0.0/16` = 65,536 possible IP addresses

- `10.240.0.0/24` = 256 addresses for my VMs

**Why I Need This:**

- Isolates cluster from other Azure resources

- Provides private network for VM-to-VM communication

- Enables custom IP address assignment

**Network Design:**

```

10.240.0.0/16 (Cluster Network - VMs)

  └─ 10.240.0.0/24 (kubernetes-subnet)

      ├─ 10.240.0.4  → jumpbox

      ├─ 10.240.0.10 → server

      ├─ 10.240.0.20 → node-0

      └─ 10.240.0.21 → node-1

10.200.0.0/16 (Pod Network - containers)

  ├─ 10.200.0.0/24 → pods on node-0

  └─ 10.200.1.0/24 → pods on node-1

```

**Why Separate Networks:**

- VM network (10.240.x.x) = infrastructure layer

- Pod network (10.200.x.x) = application layer

- Prevents IP conflicts between VMs and pods

---

### **Step 1.2-1.4: Create Cluster VMs**

**Commands:**

```bash

# Server (control plane)

az vm create \

  --name server \

  --image Ubuntu2204 \

  --size Standard_D2s_v5 \

  --vnet-name kubernetes-vnet \

  --subnet kubernetes-subnet \

  --private-ip-address 10.240.0.10 \

  --admin-username ubuntu \

  --generate-ssh-keys \

  --public-ip-address "" \

  --nsg ""

# Worker nodes (similar commands with different IPs)

az vm create --name node-0 --private-ip-address 10.240.0.20 ...

az vm create --name node-1 --private-ip-address 10.240.0.21 ...

```

**Parameter Breakdown:**

**`--name server`**

- VM hostname in Azure

- Used for Azure management (portal, CLI)

**`--image Ubuntu2204`**

- Ubuntu 22.04 LTS (Long Term Support until 2027)

- Matches KTHW tutorial default

**`--size Standard_D2s_v5`**

- 2 vCPUs, 8GB RAM

- Mixed VM families (v5, v4, v3) to work around quota limits

- All are equivalent performance for my purposes

**`--vnet-name kubernetes-vnet --subnet kubernetes-subnet`**

- Places VM in my custom network

- Enables private communication between VMs

**`--private-ip-address 10.240.0.10`**

- Static IP assignment (won't change on reboot)

- Predictable addressing for configuration

**`--admin-username ubuntu`**

- Initial user (Azure doesn't allow `root` initially)

- I'll enable root access after creation

**`--generate-ssh-keys`**

- Azure creates SSH key pair if not exists

- Stores on your Mac: `~/.ssh/id_rsa`

- Copies public key to VM

**`--public-ip-address ""`**

- No public IP (security)

- Only accessible from jumpbox (bastion pattern)

**`--nsg ""`**

- No Network Security Group

- Rely on VNet isolation for security

---

### **Step 1.5: Recreate Jumpbox in Cluster VNet**

**Why I Recreated:**

Original jumpbox was in wrong VNet (10.0.0.0/16), couldn't reach cluster VMs.

**Command:**

```bash

# Delete old jumpbox

az vm delete --name jumpbox --yes

az network nic delete --name jumpboxVMNic

az network public-ip delete --name jumpboxPublicIP

# Recreate in kubernetes-vnet

az vm create \

  --name jumpbox \

  --image Ubuntu2204 \

  --size Standard_D2s_v6 \

  --vnet-name kubernetes-vnet \

  --subnet kubernetes-subnet \

  --private-ip-address 10.240.0.4 \

  --admin-username ubuntu \

  --generate-ssh-keys \

  --public-ip-sku Standard

```

**Key Difference:** `--vnet-name kubernetes-vnet` places it in same network as cluster VMs.

**Result:**

- Private IP: 10.240.0.4 (in kubernetes-vnet)

- Can now reach all cluster VMs

**Trade-off:** Lost all previous jumpbox setup (Azure CLI, KTHW repo, etc.)

**Solution:** Reinstalled everything (see Section 2 re-setup)

---

## **PART 2: Create Machine Database**

### **Step 2.1: Create machines.txt**

**Command:**

```bash

cd ~/kubernetes-the-hard-way

cat > machines.txt <<EOF

10.240.0.10 server.kubernetes.local server

10.240.0.20 node-0.kubernetes.local node-0 10.200.0.0/24

10.240.0.21 node-1.kubernetes.local node-1 10.200.1.0/24

EOF

```

**What It Creates:**

```

IP_ADDRESS  FQDN                    HOSTNAME  POD_SUBNET

10.240.0.10 server.kubernetes.local server

10.240.0.20 node-0.kubernetes.local node-0    10.200.0.0/24

10.240.0.21 node-1.kubernetes.local node-1    10.200.1.0/24

```

**Why Each Column:**

**IP_ADDRESS:**

- Network identifier for initial access

- Used before DNS/hostnames configured

- Foundation for routing

**FQDN (Fully Qualified Domain Name):**

- Complete domain name (`server.kubernetes.local`)

- Used in TLS certificates (Section 4)

- Formal identification

**HOSTNAME:**

- Short name for convenience (`server` vs `server.kubernetes.local`)

- What appears in shell prompt

- How Kubernetes nodes register

**POD_SUBNET:**

- Only for worker nodes (server has none)

- IP range for pods on that node

- Prevents IP conflicts between nodes

**Why This File:**

1. **Single source of truth** - Change once, affects all scripts

2. **Automation** - Scripts loop through this file

3. **Documentation** - Clear record of cluster topology

4. **Scalability** - Add more machines by adding lines

---

## **PART 3: Configure SSH Access**

### **Why Root Access:**

KTHW tutorial uses `root` user throughout for simplicity. Kubernetes components need root privileges (bind to low ports, modify networking, etc.).

**Production Alternative:** Use dedicated service accounts, not root.

---

### **Step 3.1: Generate SSH Key Pair**

**Command:**

```bash

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa

```

**Parameters:**

- `-t rsa` = RSA algorithm

- `-b 4096` = 4096-bit key length (very secure)

- `-N ""` = No passphrase (empty string)

- `-f ~/.ssh/id_rsa` = Save to default location

**Creates:**

```

/home/ubuntu/.ssh/id_rsa     (Private key - SECRET!)

/home/ubuntu/.ssh/id_rsa.pub (Public key - safe to share)

```

**Why Key-Based Auth:**

- **Stronger than passwords** - 4096-bit key = effectively unbreakable

- **No password typing** - Automation-friendly

- **Revocable** - Delete public key = access denied immediately

---

### **Step 3.2: Add Jumpbox SSH Key to VMs**

**Problem:** VMs have Mac's SSH key (from `az vm create`), not jumpbox's key.

**Solution:** Use Azure CLI to add jumpbox's key:

```bash

for vm in server node-0 node-1; do

  az vm user update \

    --name ${vm} \

    --username ubuntu \

    --ssh-key-value "$(cat ~/.ssh/id_rsa.pub)"

done

```

**What This Does:**

1. Reads jumpbox's public key

2. Appends to VM's `/home/ubuntu/.ssh/authorized_keys`

3. Now jumpbox can SSH as `ubuntu` user

**Verification:**

```bash

ssh ubuntu@10.240.0.10 'echo "SSH works!"'

```

---

### **Step 3.3: Enable Root SSH**

**Commands (per VM):**

```bash

# Enable root login in SSH config

ssh ubuntu@10.240.0.10 "

  sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config

  sudo systemctl restart sshd

  sudo mkdir -p /root/.ssh

  sudo chmod 700 /root/.ssh

"

# Copy public key to root user

cat ~/.ssh/id_rsa.pub | ssh ubuntu@10.240.0.10 \

  "sudo tee -a /root/.ssh/authorized_keys"

ssh ubuntu@10.240.0.10 "sudo chmod 600 /root/.ssh/authorized_keys"

```

**What Each Command Does:**

**`sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/'`**

- `sed` = stream editor (edit files programmatically)

- `-i` = in-place edit (modify file directly)

- `s/pattern/replacement/` = substitute

- `^#*PermitRootLogin.*` = line starting with optional `#` + "PermitRootLogin"

- Replaces with: `PermitRootLogin yes`

**Before:**

```

#PermitRootLogin prohibit-password

```

**After:**

```

PermitRootLogin yes

```

**`systemctl restart sshd`**

- Restarts SSH daemon

- Loads new configuration from disk into memory

**`mkdir -p /root/.ssh && chmod 700 /root/.ssh`**

- Creates `.ssh` directory for root user

- `chmod 700` = only root can read/write/execute

**`cat ~/.ssh/id_rsa.pub | ssh ... "sudo tee -a /root/.ssh/authorized_keys"`**

- Reads jumpbox's public key

- Pipes to remote command

- `tee -a` = append to file (with sudo privileges)

**`chmod 600 /root/.ssh/authorized_keys`**

- Only root can read/write (SSH requirement)

**Why This Works:**

1. SSH daemon now allows root login

2. Root's authorized_keys contains jumpbox's public key

3. Jumpbox can now SSH as root without password

---

### **Step 3.4: Verify Root Access**

**Command:**

```bash

while read IP FQDN HOST SUBNET; do

  ssh -n root@${IP} hostname

done < machines.txt

```

**Expected Output:**

```

server

node-0

node-1

```

**Flag Explanation:**

**`-n` flag (CRITICAL for loops):**

**Without `-n`:**

```bash

while read line; do

  ssh user@host command

done < file.txt

# SSH consumes remaining lines from file.txt!

# Loop only processes first line

```

**With `-n`:**

```bash

while read line; do

  ssh -n user@host command

done < file.txt

# SSH ignores stdin

# Loop processes all lines correctly

```

**Why It Matters:**

SSH normally reads from stdin. In a loop reading from a file, SSH would consume the file's remaining lines, breaking the loop.

---

## **PART 4: Configure Hostnames**

### **Why Set Hostnames:**

**Three Reasons:**

1. **Human Readability**

   - Shell prompt: `root@server:~#` vs `root@ip-10-240-0-10:~#`

   - Clear which machine you're on

2. **Kubernetes Node Registration**

   - Nodes register with API server using their hostname

   - Shows as `server`, `node-0`, `node-1` in `kubectl get nodes`

3. **Consistency**

   - Hostname matches what I'll reference in configs

   - Hostname matches DNS entries

---

### **Step 4.1: Set Hostname on Each Machine**

**Command:**

```bash

while read IP FQDN HOST SUBNET; do

  ssh -n root@${IP} "hostnamectl set-hostname ${HOST}"

  ssh -n root@${IP} "systemctl restart systemd-hostnamed"

done < machines.txt

```

**What `hostnamectl` Does:**

1. Updates `/etc/hostname` file

2. Updates kernel hostname (`uname -n`)

3. Updates systemd-hostnamed service

4. Changes shell prompt

**Persistent vs Temporary:**

```bash

# Temporary (resets on reboot):

hostname server

# Persistent (survives reboot):

hostnamectl set-hostname server

```

**Why Restart systemd-hostnamed:**

- Ensures all systemd services see the new hostname

- Propagates change across the system

---

### **Step 4.2: Update /etc/hosts Loopback Entry**

**Command:**

```bash

while read IP FQDN HOST SUBNET; do

  ssh -n root@${IP} "sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"

done < machines.txt

```

**What It Changes:**

**Before:**

```

127.0.1.1 ip-10-240-0-10

```

**After:**

```

127.0.1.1 server.kubernetes.local server

```

**Why 127.0.1.1:**

- `127.0.0.1` = localhost (never changes)

- `127.0.1.1` = system hostname (Ubuntu convention)

- Maps loopback IP to both FQDN and short hostname

**Why This Matters:**

Programs that look up the local hostname get consistent results.

---

## **PART 5: Configure DNS Resolution**

### **Why DNS Resolution:**

**The Problem Without DNS:**

```bash

ssh root@10.240.0.20  # Hard to remember which node this is

ping 10.240.0.10      # Is this server or node-0?

```

**With DNS:**

```bash

ssh root@node-0       # Clear, readable

ping server           # Obvious

```

**How It Works:**

```

User types: ssh root@server

          ↓

System checks: /etc/hosts

          ↓

Finds: 10.240.0.10 server.kubernetes.local server

          ↓

Connects to: 10.240.0.10

```

---

### **Step 5.1: Create Hosts File**

**Command:**

```bash

cat > hosts <<EOF

# Kubernetes The Hard Way

10.240.0.10 server.kubernetes.local server

10.240.0.20 node-0.kubernetes.local node-0

10.240.0.21 node-1.kubernetes.local node-1

EOF

```

**What It Creates:**

Simple text file with IP → hostname mappings.

**Why Separate File:**

- Can distribute to all machines

- Version control friendly

- Easy to edit/update

---

### **Step 5.2: Add to Jumpbox /etc/hosts**

**Command:**

```bash

cat hosts | sudo tee -a /etc/hosts

```

**Why `tee -a`:**

- `cat hosts >> /etc/hosts` would fail (permission denied)

- `sudo cat hosts >> /etc/hosts` doesn't work (redirect happens before sudo)

- `tee -a` reads stdin and appends with proper permissions

**Verification:**

```bash

tail /etc/hosts

```

**Should show:**

```

# Kubernetes The Hard Way

10.240.0.10 server.kubernetes.local server

10.240.0.20 node-0.kubernetes.local node-0

10.240.0.21 node-1.kubernetes.local node-1

```

---

### **Step 5.3: Test from Jumpbox**

**Command:**

```bash

ping -c 2 server

```

**What Happens:**

1. System reads `/etc/hosts`

2. Finds: `10.240.0.10 server.kubernetes.local server`

3. Resolves `server` → `10.240.0.10`

4. Sends ICMP packets to 10.240.0.10

**Success Output:**

```

PING server.kubernetes.local (10.240.0.10) 56(84) bytes of data.

64 bytes from server.kubernetes.local (10.240.0.10): icmp_seq=1 ttl=64 time=5.10 ms

```

**Notice:** "PING server.kubernetes.local" - shows it resolved to FQDN!

---

### **Step 5.4: Distribute to Cluster VMs**

**Command:**

```bash

while read IP FQDN HOST SUBNET; do

  scp hosts root@${IP}:~/

  ssh -n root@${IP} "cat hosts >> /etc/hosts"

done < machines.txt

```

**What Each Command Does:**

**`scp hosts root@${IP}:~/`**

- Secure Copy (encrypted)

- Copies `hosts` file to VM's `/root/` directory

- Uses SSH for transfer

**`ssh -n root@${IP} "cat hosts >> /etc/hosts"`**

- Appends `hosts` file to VM's `/etc/hosts`

- Now VM can resolve all hostnames

**Why Distribute:**

- Each VM needs its own copy of /etc/hosts

- VMs need to resolve each other's hostnames

- Example: node-0 needs to find server, node-1

---

### **Step 5.5: Verify Cross-VM DNS**

**Commands:**

```bash

ssh root@server 'ping -c 2 node-0'

ssh root@node-0 'ping -c 2 server'

ssh root@node-1 'ping -c 2 node-0'

```

**What I'm Testing:**

- server → node-0 (control plane to worker)

- node-0 → server (worker to control plane)

- node-1 → node-0 (worker to worker)

**Why This Matters:**

Future sections require:

- API server (on server) talking to kubelets (on workers)

- Workers communicating for pod networking

- All components finding each other by name

---

## **PART 6: Final Verification**

### **Complete Connectivity Matrix**

**Command:**

```bash

ssh root@server 'ping -c 1 node-0 > /dev/null && echo "✓ server → node-0"'

ssh root@server 'ping -c 1 node-1 > /dev/null && echo "✓ server → node-1"'

ssh root@node-0 'ping -c 1 server > /dev/null && echo "✓ node-0 → server"'

ssh root@node-0 'ping -c 1 node-1 > /dev/null && echo "✓ node-0 → node-1"'

ssh root@node-1 'ping -c 1 server > /dev/null && echo "✓ node-1 → server"'

ssh root@node-1 'ping -c 1 node-0 > /dev/null && echo "✓ node-1 → node-0"'

```

**What I Verified:**

```

     ┌─────────┐

  ┌──┤  server │──┐

  │  └─────────┘  │

  │               │

  ↓               ↓

┌───────┐     ┌───────┐

│node-0 │←────│node-1 │

└───────┘     └───────┘

```

**All connections bidirectional** ✓

---

## **Section 3 Accomplishments**

### **Infrastructure:**

- ✅ 4 VMs created (1 jumpbox, 1 server, 2 workers)

- ✅ All in kubernetes-vnet (10.240.0.0/24)

- ✅ Static IP addresses assigned

- ✅ Jumpbox has public IP for external access

- ✅ Cluster VMs private (security)

### **Networking:**

- ✅ Full mesh connectivity verified

- ✅ DNS resolution works (hostnames → IPs)

- ✅ VMs can find each other by name

### **Access:**

- ✅ SSH key authentication configured

- ✅ Root access enabled on all cluster VMs

- ✅ Jumpbox can SSH to any VM by hostname

### **Identity:**

- ✅ Hostnames set and persistent

- ✅ FQDNs configured

- ✅ /etc/hosts entries distributed

---

## **What I Learned**

### **1. Cloud Quotas Are Real**

New accounts have tight limits. Always check before provisioning.

### **2. Network Design Matters**

VMs must be in same VNet to communicate. Plan network topology upfront.

### **3. Security Trade-offs**

Enabled root SSH for convenience (learning environment). Production: use service accounts.

### **4. Automation Patterns**

- Loop through machine database

- Use `-n` flag with SSH in loops

- Single source of truth (machines.txt)

### **5. DNS Is Foundation**

Name resolution enables everything else. IP addresses are implementation details.

---

## **Azure vs KTHW Tutorial Differences**

| Aspect | KTHW Tutorial | My Azure Setup |

|--------|---------------|-----------------|

| **VMs** | Local (Multipass/VirtualBox) | Azure VMs |

| **Networking** | Host-only network | Azure VNet |

| **Cost** | Free | ~£15/week |

| **Initial Username** | Can use root | Must use ubuntu, then enable root |

| **Network Config** | Automatic | Manual VNet creation |

| **VM Creation** | `multipass launch` | `az vm create` |

| **SSH Keys** | Manual generation | Azure auto-generates |

| **Quota** | Limited by RAM | Limited by cloud quota |

---

## **Commands Reference**

### **VM Management:**

```bash

# List VMs

az vm list -d -o table

# Start/stop VM

az vm start --name server

az vm deallocate --name jumpbox

# Delete VM

az vm delete --name server --yes

```

### **SSH:**

```bash

# SSH by IP

ssh root@10.240.0.10

# SSH by hostname (after /etc/hosts configured)

ssh root@server

# SSH with strict host key checking disabled

ssh -o StrictHostKeyChecking=no root@server

# SSH without consuming stdin (in loops)

ssh -n root@server command

```

### **Network Testing:**

```bash

# Test connectivity

ping -c 2 server

# Test from remote VM

ssh root@server 'ping -c 1 node-0'

```

---

## **Troubleshooting**

### **Issue: SSH connection timeout**

**Cause:** VMs in different VNets

**Check:**

```bash

az vm show --name jumpbox --query "networkProfile.networkInterfaces[0].id" -o tsv | \

  xargs az network nic show --ids | grep vnet

```

**Fix:** Recreate VM in correct VNet

---

### **Issue: Permission denied (publickey)**

**Cause:** SSH key not on target VM

**Fix:**

```bash

# Add key via Azure CLI

az vm user update --name server --username root \

  --ssh-key-value "$(cat ~/.ssh/id_rsa.pub)"

```

---

### **Issue: Quota exceeded**

**Cause:** Insufficient vCPU quota

**Check:**

```bash

az vm list-usage --location westeurope -o table | grep vCPU

```

**Fix:** Upgrade account or request quota increase

---

### **Issue: "Could not resolve hostname server"**

**Cause:** /etc/hosts not configured

**Check:**

```bash

cat /etc/hosts | grep server

```

**Fix:** Rerun DNS configuration steps

---

## **Cost Tracking**

**Current Usage:**

- jumpbox: Standard_D2s_v6 (2 vCPU, 8GB RAM) = ~£0.10/hour

- server: Standard_D2s_v5 (2 vCPU, 8GB RAM) = ~£0.10/hour

- node-0: Standard_D2s_v4 (2 vCPU, 8GB RAM) = ~£0.10/hour

- node-1: Standard_D2s_v3 (2 vCPU, 8GB RAM) = ~£0.10/hour

**Total:** ~£0.40/hour = ~£9.60/day

**For 1 week:** ~£67

**Cost Saving Tips:**

```bash

# Deallocate when not using (stops billing)

az vm deallocate --name jumpbox --name server --name node-0 --name node-1

# Start when ready to resume

az vm start --name jumpbox --name server --name node-0 --name node-1

```

---

## **Next: Section 4 - Certificate Authority**

Now that I have:

- ✅ Working VMs

- ✅ Network connectivity

- ✅ SSH access

- ✅ Name resolution

**I'm ready to:**

1. Generate TLS certificates for cluster security

2. Create Certificate Authority (root of trust)

3. Issue certificates for each component

4. Distribute certificates to machines

All certificates will use the hostnames I configured (server.kubernetes.local, etc.).

---

## **Key Takeaways**

1. **Infrastructure as Code:** machines.txt = single source of truth

2. **Security Layers:** Network isolation + SSH keys + (future: TLS certs)

3. **Name Resolution:** Foundation for everything else

4. **Automation:** Loops + SSH = scalable configuration

5. **Cloud Constraints:** Quotas, naming rules, network topology matter