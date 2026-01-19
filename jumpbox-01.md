# KTHW on Azure - Section 2: Setting up the Jumpbox

**KTHW Tutorial:** https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-jumpbox.md

---

## Table of Contents
- [Conceptual Understanding](#conceptual-understanding)
- [Jumpbox vs Direct Access](#jumpbox-vs-direct-access)
- [Creating the Jumpbox on Azure](#creating-the-jumpbox-on-azure)
- [Connecting to the Jumpbox](#connecting-to-the-jumpbox)
- [Installing Tools on the Jumpbox](#installing-tools-on-the-jumpbox)
- [Downloading Kubernetes Binaries](#downloading-kubernetes-binaries)
- [Jumpbox Role in KTHW](#jumpbox-role-in-kthw)
- [Network Topology](#network-topology-after-section-2)
- [Key Concepts Learned](#key-concepts-learned)
- [Troubleshooting](#troubleshooting)

---

## Conceptual Understanding

### What is a Jumpbox?

**Definition:**

A jumpbox (also called bastion host) is a special-purpose VM that acts as a secure gateway to private infrastructure.

**Why Use a Jumpbox?**

```

Your Mac (Public Internet)

    ↓ SSH

Jumpbox (Public IP + Private IP)

    ↓ SSH (private network)

Cluster VMs (Private IPs only)

```

**Security Benefits:**

1. **Single entry point** - Only jumpbox exposed to internet

2. **Audit trail** - All access goes through jumpbox (can log)

3. **Reduced attack surface** - Cluster VMs have no public IPs

4. **Defense in depth** - Attacker must compromise jumpbox first

**Production Pattern:**

- AWS: EC2 bastion host in public subnet

- Azure: Jumpbox VM with NSG rules

- GCP: IAP (Identity-Aware Proxy) for SSH

---

## Jumpbox vs Direct Access

| Aspect | With Jumpbox | Without Jumpbox |

|--------|-------------|----------------|

| **Security** | High (single entry point) | Low (all VMs exposed) |

| **Cost** | +1 VM | 0 extra VMs |

| **Convenience** | Two SSH hops | One SSH hop |

| **Compliance** | Meets most standards | Often non-compliant |

| **Blast Radius** | Contained | Full cluster exposed |

---

## Creating the Jumpbox on Azure

### Command

```bash

az vm create \

  --resource-group kthw-learning \

  --name jumpbox \

  --image Ubuntu2204 \

  --size Standard_D2s_v6 \

  --admin-username ubuntu \

  --generate-ssh-keys \

  --public-ip-sku Standard \

  --location westeurope

```

### Parameter Breakdown

**`--resource-group kthw-learning`**

- Places VM in my KTHW resource group

- Enables bulk deletion later

**`--name jumpbox`**

- VM hostname

- DNS name will be `jumpbox` (resolvable within Azure VNet)

- Externally referred to by public IP

**`--image Ubuntu2204`**

- Ubuntu 22.04 LTS

- Same as KTHW tutorial default

- Long-term support until 2027

**`--size Standard_D2s_v6`**

- 2 vCPU, 8GB RAM

- More than sufficient for jumpbox role

- Could use smaller size (D1s) to save cost

**`--admin-username ubuntu`**

- Default user account

- Matches KTHW tutorial convention

- Gets sudo privileges automatically

**`--generate-ssh-keys`**

- Azure creates SSH key pair if not exists

- Stores in: `~/.ssh/id_rsa`

- Copies public key to VM: `~/.ssh/authorized_keys`

**`--public-ip-sku Standard`**

- Allocates public IP address

- "Standard" SKU (vs "Basic") for production features

- Enables SSH from internet

**`--location westeurope`**

- Same region as resource group

- Lower latency to cluster VMs (same datacenter)

### Output

```json

{

  "fqdns": "",

  "id": "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/kthw-learning/providers/Microsoft.Compute/virtualMachines/jumpbox",

  "location": "westeurope",

  "powerState": "VM running",

  "privateIpAddress": "<PRIVATE_IP>",

  "publicIpAddress": "<PUBLIC_IP>",

  "resourceGroup": "kthw-learning"

}

```

**Key Information:**

- **Public IP:** Assigned by Azure (for SSH access)

- **Private IP:** `10.0.0.4` (within Azure VNet)

- **Power State:** `VM running` (ready to use)

**Note on Private IP:**

- Azure auto-assigned `10.0.0.4` (I didn't specify)

- Created default VNet: `10.0.0.0/16`

- I'll create separate VNet for cluster (10.240.0.0/16)

---

## Connecting to the Jumpbox

### From Your Mac

```bash

ssh ubuntu@<PUBLIC_IP>

```

**What Happens:**

1. SSH client looks for private key (`~/.ssh/id_rsa`)

2. Connects to public IP over internet

3. Azure VM verifies public key matches

4. Shell session starts as `ubuntu` user

**First Connection:**

```

The authenticity of host '<PUBLIC_IP>' can't be established.

ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.

Are you sure you want to continue connecting (yes/no)? yes

```

This is normal! SSH is verifying you're connecting to the intended host.

---

## Installing Tools on the Jumpbox

### 1. Update Package Index

```bash

sudo apt-get update

```

**What This Does:**

- Refreshes list of available packages

- Gets latest security patches

- Doesn't install anything yet

**Why:**

- Ubuntu package lists can be stale on new VMs

- Ensures I get current versions

---

### 2. Install Command Line Utilities

```bash

sudo apt-get -y install wget curl vim openssl git

```

**Tools Installed:**

**`wget`**

- Downloads files from URLs

- Used to fetch Kubernetes binaries

- Supports resume, recursive downloads

**`curl`**

- Transfer data to/from servers

- Used for API calls, downloading scripts

- More versatile than wget

**`vim`**

- Text editor (vi improved)

- Edit configuration files

- KTHW tutorial assumes vim knowledge

**`openssl`**

- Cryptography toolkit

- Generate certificates (Section 4)

- Inspect certs, test TLS connections

**`git`**

- Version control

- Clone KTHW repository

- Track changes to configs

**`-y` flag:**

- Auto-answer "yes" to prompts

- Non-interactive installation

---

### 3. Install Azure CLI on Jumpbox

**Why Install Azure CLI on Jumpbox?**

- Create cluster VMs from jumpbox (not local Mac)

- Jumpbox is the control center for cluster operations

- Easier than creating VMs from Mac, then SSHing in

```bash

curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

```

**What This Does:**

1. Downloads Microsoft's installation script

2. Adds Azure CLI apt repository

3. Installs `azure-cli` package

**Flags:**

- `-s`: Silent mode (no progress bar)

- `-L`: Follow redirects

---

### 4. Login to Azure from Jumpbox

```bash

az login

```

**Device Code Flow:**

```

To sign in, use a web browser to open the page https://microsoft.com/devicelogin

and enter the code DG9CNYLVX to authenticate.

```

**Why Device Code?**

- Jumpbox has no web browser

- Opens browser on your Mac

- Enters code to link session

**Alternative (if you have credentials):**

```bash

az login --use-device-code  # Explicit device code flow

az login -u user@domain.com -p password  # Not recommended (insecure)

az login --service-principal  # For CI/CD automation

```

---

### 5. Configure Azure Defaults

```bash

az configure --defaults group=kthw-learning location=westeurope

```

**What This Does:**

- Sets default resource group for all commands

- Sets default region for resource creation

- Saves typing, prevents mistakes

**Without defaults:**

```bash

az vm create --resource-group kthw-learning --location westeurope --name test

```

**With defaults:**

```bash

az vm create --name test

```

---

## Downloading Kubernetes Binaries

### 1. Clone KTHW Repository

```bash

git clone --depth 1 \

  https://github.com/kelseyhightower/kubernetes-the-hard-way.git

cd kubernetes-the-hard-way

```

**`--depth 1`:**

- Shallow clone (only latest commit)

- Faster download

- Smaller disk usage (~10MB vs ~50MB)

**Repository Structure:**

```

kubernetes-the-hard-way/

├── docs/               # Tutorial chapters

├── downloads.txt       # URLs for binaries (ARM64)

├── downloads-amd64.txt # URLs for binaries (AMD64)

└── configs/            # Sample config files

```

---

### 2. Download Binaries

```bash

wget -q --show-progress \

  --https-only \

  --timestamping \

  -P downloads \

  -i downloads-$(dpkg --print-architecture).txt

```

**Parameter Breakdown:**

**`-q --show-progress`**

- Quiet mode (less verbose)

- Show progress bar only

- Clean output

**`--https-only`**

- Only download via HTTPS

- Security: prevents man-in-the-middle

- Ensures binary integrity

**`--timestamping`**

- Only download if remote file is newer

- Enables resume on failure

- Idempotent (can re-run safely)

**`-P downloads`**

- Save files to `downloads/` directory

- Organizes binaries

**`-i downloads-$(dpkg --print-architecture).txt`**

- Read URLs from file

- `$(dpkg --print-architecture)` = detects CPU arch (amd64 or arm64)

- Downloads correct binaries for your CPU

**Downloaded Files (~566MB):**

```

cni-plugins-linux-amd64-v1.6.2.tgz       # Container networking

containerd-2.1.0-beta.0-linux-amd64.tar.gz  # Container runtime

crictl-v1.32.0-linux-amd64.tar.gz        # Container runtime CLI

etcd-v3.6.0-rc.3-linux-amd64.tar.gz      # Distributed key-value store

kube-apiserver                            # K8s API server binary

kube-controller-manager                   # K8s controller manager

kube-proxy                                # K8s network proxy

kube-scheduler                            # K8s scheduler

kubectl                                   # K8s CLI client

kubelet                                   # K8s node agent

runc.amd64                                # Low-level container runtime

```

---

### 3. Extract and Organize Binaries

```bash

{

  ARCH=$(dpkg --print-architecture)

  mkdir -p downloads/{client,cni-plugins,controller,worker}

  # Extract to appropriate directories

  tar -xvf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz \

    -C downloads/worker/

  tar -xvf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz \

    --strip-components 1 \

    -C downloads/worker/

  tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.6.2.tgz \

    -C downloads/cni-plugins/

  tar -xvf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz \

    -C downloads/ \

    --strip-components 1 \

    etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl \

    etcd-v3.6.0-rc.3-linux-${ARCH}/etcd

  # Move binaries to correct locations

  mv downloads/{etcdctl,kubectl} downloads/client/

  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \

    downloads/controller/

  mv downloads/{kubelet,kube-proxy} downloads/worker/

  mv downloads/runc.${ARCH} downloads/worker/runc

}

```

**Directory Structure After Extraction:**

```

downloads/

├── client/

│   ├── etcdctl           # etcd CLI (for debugging/backup)

│   └── kubectl           # K8s CLI

├── cni-plugins/

│   ├── bridge            # CNI plugin for pod networking

│   ├── loopback          # Loopback interface

│   ├── host-local        # IP address management

│   └── ...               # 14 total plugins

├── controller/

│   ├── etcd              # Key-value store

│   ├── kube-apiserver    # API server

│   ├── kube-controller-manager  # Controllers

│   └── kube-scheduler    # Scheduler

└── worker/

    ├── containerd        # Container runtime

    ├── crictl            # Container runtime CLI

    ├── kubelet           # Node agent

    ├── kube-proxy        # Network proxy

    └── runc              # Low-level runtime

```

**Why This Organization?**

- **client/**: Tools for jumpbox (kubectl, etcdctl)

- **controller/**: Control plane binaries (install on `server` VM)

- **worker/**: Worker node binaries (install on `node-0`, `node-1`)

- **cni-plugins/**: Networking (install on workers)

**tar flags:**

- `-x`: Extract

- `-v`: Verbose (show files)

- `-f`: File to extract

- `-C`: Change to directory

- `--strip-components 1`: Remove top-level directory from archive

---

### 4. Make Binaries Executable

```bash

chmod +x downloads/{client,cni-plugins,controller,worker}/*

```

**Why:**

- Tarballs don't preserve execute permissions

- Binaries must be executable to run

- `chmod +x` adds execute permission

**Verification:**

```bash

ls -lh downloads/client/

```

Output:

```

-rwxr-xr-x 1 ubuntu ubuntu 55M kubectl

-rwxr-xr-x 1 ubuntu ubuntu 23M etcdctl

```

`x` in permissions = executable ✓

---

### 5. Install kubectl Globally

```bash

sudo cp downloads/client/kubectl /usr/local/bin/

```

**Why `/usr/local/bin/`?**

- In `$PATH` (can run `kubectl` from anywhere)

- System-wide installation (all users can use)

- Convention for manually installed binaries

**Verification:**

```bash

kubectl version --client

```

Output:

```

Client Version: v1.32.3

Kustomize Version: v5.5.0

```

✓ kubectl installed successfully!

---

## Jumpbox Role in KTHW

**What I'll Do from Jumpbox:**

1. **Section 3**: Create cluster VMs (server, node-0, node-1)

2. **Section 4**: Generate TLS certificates for all components

3. **Section 5**: Create kubeconfig files for authentication

4. **Section 6**: Generate encryption key for secrets

5. **Sections 7-9**: Distribute binaries/configs to cluster VMs via SSH

6. **Section 10**: Configure kubectl for remote cluster access

7. **Section 11**: Set up pod network routing

8. **Section 12**: Run smoke tests

9. **Section 13**: Clean up resources

**Jumpbox = Your Mission Control**

---

## Network Topology After Section 2

```

Internet

    ↓

Jumpbox (10.0.0.4)

├─ Public IP: <PUBLIC_IP>

├─ VNet: 10.0.0.0/16 (Azure auto-created)

└─ SSH access to: [cluster VMs will go in separate VNet in Section 3]

```

**Next Section:**

I'll create a separate VNet (10.240.0.0/16) for the cluster VMs.

---

## Key Concepts Learned

### 1. Bastion Host Pattern

- Security best practice in cloud environments

- Used by enterprises (banks, government, etc.)

- Alternative: VPN (more complex)

### 2. Binary Distribution

- Kubernetes components are standalone binaries

- No package manager (apt/yum) for K8s

- I download official releases from GitHub/Google Cloud

### 3. Multi-Architecture Support

- `dpkg --print-architecture` detects CPU (amd64/arm64)

- Kubernetes releases for both architectures

- Important for M-series Macs (ARM64)

### 4. Infrastructure as Code Foundations

- `az vm create` = declarative infrastructure

- Can script entire setup (future: Terraform version)

- Reproducible environments

---

## Troubleshooting

**Issue: Can't SSH to jumpbox**

```bash

ssh -v ubuntu@<PUBLIC_IP>  # Verbose mode shows details

```

- Check: Public IP correct?

- Check: SSH key exists? `ls ~/.ssh/id_rsa`

- Check: Azure Network Security Group allows port 22?

**Issue: `wget` download fails**

```bash

wget --no-check-certificate URL  # Bypass SSL verification (debug only!)

```

- Check: Internet connectivity from jumpbox?

- Check: GitHub accessible? `ping github.com`

**Issue: `tar` extraction fails**

```bash

tar -tvf file.tar.gz  # List contents without extracting

```

- Check: Download completed? `ls -lh downloads/`

- Check: File not corrupted? Re-download

**Issue: `kubectl: command not found`**

```bash

which kubectl  # Check if in PATH

echo $PATH     # View PATH directories

```

- Check: Copied to `/usr/local/bin/`?

- Check: `/usr/local/bin/` in PATH?

---

## Cost Tracking

**Jumpbox VM Cost:**

- Size: Standard_D2s_v6

- Rate: ~£0.10/hour

- Daily: ~£2.40 (24 hours)

- Weekly: ~£16.80

**How to Check:**

```bash

az consumption usage list \

  --start-date 2025-12-28 \

  --end-date 2025-12-29 \

  -o table

```

**Stop VM When Not Using:**

```bash

# From your Mac:

az vm deallocate --name jumpbox --resource-group kthw-learning

# Start it again:

az vm start --name jumpbox --resource-group kthw-learning

```

**Deallocate vs Stop:**

- **Deallocate**: Releases compute resources, stops billing ✓

- **Stop** (from VM): VM stopped but still allocated, continues billing ✗

---

## What's Different from KTHW Tutorial?

| KTHW Tutorial | My Azure Setup |

|---------------|----------------|

| Multipass VM on local machine | Azure VM in cloud |

| No additional cost | £0.10/hour |

| Local-only access | Internet-accessible via public IP |

| Manual IP assignment | Azure DHCP + static private IP |

| Single subnet | Default VNet (will create custom VNet) |

---

## Section 2 Completion Checklist

- [x] Jumpbox VM created on Azure

- [x] Public IP obtained

- [x] SSH access verified from Mac

- [x] Azure CLI installed on jumpbox

- [x] Logged into Azure from jumpbox

- [x] Default resource group configured

- [x] KTHW repository cloned

- [x] Kubernetes binaries downloaded (~566MB)

- [x] Binaries extracted and organized

- [x] Binaries made executable

- [x] kubectl installed globally

- [x] kubectl version verified (v1.32.3)

---

## Next: Section 3 - Provisioning Compute Resources

I'll create 3 VMs for the Kubernetes cluster:

- **server**: Control plane (10.240.0.10)

- **node-0**: Worker (10.240.0.20)

- **node-1**: Worker (10.240.0.21)