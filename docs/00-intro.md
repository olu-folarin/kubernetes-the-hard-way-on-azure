# KTHW on Azure - Section 0: Initial Setup

**Purpose:** Document Azure-specific setup before starting KTHW tutorial

---

## Why Azure Instead of Local?

**Decision Rationale:**

1. **Learning Azure** - Unfamiliar with Azure, want hands-on experience

2. **Hardware Constraints** - M2 Mac with 8GB RAM too limited for 4 VMs locally

3. **Dual Learning Objective** - Learn Kubernetes + Azure simultaneously

**Note:** New Azure accounts have VM size restrictions - B-series may be unavailable, D-series v6 works.

---

## Azure CLI Installation (Local Mac)

**Installation:**

```bash

brew install azure-cli

```

**Verification:**

```bash

az --version

```

**Login:**

```bash

az login

# Opens browser for authentication

# Confirms subscription: Azure subscription 1

```

---

## Resource Group Creation

**Purpose:** Logical container for all KTHW resources in Azure

**Command:**

```bash

az group create \

  --name kthw-learning \

  --location westeurope

```

**Why `westeurope`?**

- Initial attempt used `uksouth` → Standard_B2s not available

- Tried `westeurope` → Standard_B2s also restricted (new account limits)

- Checked available VM sizes with:

  ```bash

  az vm list-skus \

    --location westeurope \

    --resource-type virtualMachines \

    --output table | grep -v "NotAvailableForSubscription"

  ```

- Found Standard_D2s_v6 available (2 vCPU, 8GB RAM)

**Key Learning:**

- Always check `az vm list-skus` before attempting VM creation

- Saves time vs trial-and-error with `az vm create`

**Resource Group:** `kthw-learning` in `westeurope`

---

## VM Size Selection Process

**Trial 1: Standard_B2s (Failed)**

- Attempted in `uksouth` → Capacity restrictions

- Attempted in `westeurope` → NotAvailableForSubscription

**Trial 2: Standard_D2s_v5 (Failed)**

- Azure CLI recommended as new default

- NotAvailableForSubscription (new account restriction)

**Trial 3: Standard_D2s_v3 (Failed)**

- Older generation, still restricted

**Success: Standard_D2s_v6**

- Latest generation (v6 series)

- 2 vCPU, 8GB RAM

- Available in westeurope zones 1,2,3

- Suitable for KTHW requirements (minimum 2 vCPU, 2GB RAM)

**VM Size Specifications:**

| Component | Spec |

|-----------|------|

| Model | Standard_D2s_v6 |

| vCPUs | 2 |

| RAM | 8GB |

| Storage | Premium SSD supported |

| Cost | ~£0.10-0.12/hour |

---

## Network Topology Design

**Cluster Network (VMs):**

```

10.240.0.0/16 (65,536 addresses)

  └─ 10.240.0.0/24 (256 addresses) - kubernetes-subnet

      ├─ 10.240.0.4  → jumpbox (bastion host)

      ├─ 10.240.0.10 → server (control plane) [to be created]

      ├─ 10.240.0.20 → node-0 (worker) [to be created]

      └─ 10.240.0.21 → node-1 (worker) [to be created]

```

**Pod Network (Application Layer):**

```

10.200.0.0/16 (65,536 addresses)

  ├─ 10.200.0.0/24 → pods running on node-0

  └─ 10.200.1.0/24 → pods running on node-1

```

**Why These Ranges?**

- `10.x.x.x` = RFC 1918 private address space

- `/16` for cluster = room for 256 /24 subnets

- `/24` per node for pods = 254 pods per node

- Separate ranges prevent IP conflicts between VMs and pods

**Network Isolation:**

- Jumpbox: Public IP (for SSH access) + Private IP (for cluster access)

- Cluster VMs: Private IPs only (no direct internet access)

- Security: Jumpbox acts as bastion, cluster isolated

---

## Lessons Learned

1. **Check availability first** - `az vm list-skus` saves 30+ minutes of errors

2. **New accounts have restrictions** - Can't assume any VM size works

3. **Latest generation sometimes more available** - v6 worked when v3/v5 didn't

4. **Region matters** - Availability varies by region

5. **Cost calculation** - VMs charged per hour running, not just existing

---

## Next Steps

After this setup:

- Section 1: Prerequisites (verify tools on jumpbox)

- Section 2: Jumpbox setup (install K8s binaries)

- Section 3: Provision compute resources (create cluster VMs)

---

## References

- Azure CLI docs: https://learn.microsoft.com/en-us/cli/azure/

- VM size restrictions: https://aka.ms/azureskunotavailable

- KTHW original: https://github.com/kelseyhightower/kubernetes-the-hard-way