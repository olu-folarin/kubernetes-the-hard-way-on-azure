# KTHW on Azure - Section 13: Tear Down Infrastructure

---

## **Section Overview**

**Goal:** Delete all Azure resources to stop incurring charges.

**Important:** Run these commands from your **LOCAL machine**, not the jumpbox - you'll lose access once deletion starts.

---

## **Why This Matters**

### **Cost Implications**

Azure resources incur charges while running:
- VMs (jumpbox, server, node-0, node-1)
- Storage (OS disks)
- Network resources (VNet, public IPs)

Deleting the resource group removes **everything** created for KTHW and stops all charges.

---

## **PART 1: Verify Resource Group**

### **Step 1.1: List Resource Groups**

**Command:**
```bash
az group list --output table
```

**What It Does:**
Lists all resource groups in your Azure subscription.

**Expected Output:**
```
Name           Location    Status
-------------  ----------  ---------
kthw-learning  westeurope  Succeeded
```

**Why This Step:**
Confirms the correct resource group name before deletion - prevents accidentally deleting the wrong resources.

---

## **PART 2: Delete Everything**

### **Step 2.1: Delete Resource Group**

**Command:**
```bash
az group delete --name kthw-learning --yes --no-wait
```

**What It Does:**
Deletes the resource group and all resources inside it.

**Analogy:**
Like throwing away an entire filing cabinet instead of removing papers one by one.

**Parameter Breakdown:**

| Parameter | Purpose |
|-----------|---------|
| `--name kthw-learning` | Target resource group |
| `--yes` | Skip confirmation prompt |
| `--no-wait` | Return immediately (don't wait for deletion) |

**What Gets Deleted:**
```
kthw-learning/
├── jumpbox VM + disk + NIC + public IP
├── server VM + disk + NIC
├── node-0 VM + disk + NIC
├── node-1 VM + disk + NIC
├── kubernetes-vnet
├── kubernetes-subnet
└── All associated resources
```

**Timeline:**
- Command returns: Immediately (with `--no-wait`)
- Actual deletion: 5-10 minutes in background

---

## **PART 3: Monitor Progress**

### **Step 3.1: Check Deletion Status**

**Command:**
```bash
az group show --name kthw-learning
```

**While Deleting:**
```json
{
  "name": "kthw-learning",
  "properties": {
    "provisioningState": "Deleting"
  }
}
```

**When Complete:**
```
(ResourceGroupNotFound) Resource group 'kthw-learning' could not be found.
```

**What This Means:**
Once you get the "not found" error, everything is deleted and charges have stopped.

---

## **Verification**

### **Confirm No Resources Remain**

**Command:**
```bash
az group list --output table
```

**Expected:** `kthw-learning` should not appear in the list.

### **Check Azure Portal (Optional)**

1. Go to [portal.azure.com](https://portal.azure.com)
2. Navigate to Resource Groups
3. Verify `kthw-learning` no longer exists

---

## **What Was Deleted**

### **Compute Resources**
| Resource | Type | Purpose |
|----------|------|---------|
| jumpbox | VM | Bastion host for cluster access |
| server | VM | Kubernetes control plane |
| node-0 | VM | Worker node |
| node-1 | VM | Worker node |

### **Network Resources**
| Resource | Type | Purpose |
|----------|------|---------|
| kubernetes-vnet | Virtual Network | Cluster network (10.240.0.0/16) |
| kubernetes-subnet | Subnet | Node subnet (10.240.0.0/24) |
| jumpbox-pip | Public IP | SSH access to jumpbox |

### **Storage Resources**
| Resource | Type | Purpose |
|----------|------|---------|
| *_OsDisk_* | Managed Disk | OS disks for each VM |

---

## **Cost Summary**

### **Estimated Costs During KTHW:**

| Resource | Approx. Cost/Hour |
|----------|-------------------|
| 4x Standard_D2s VMs | ~$0.40/hour total |
| Storage | ~$0.01/hour |
| Network | Minimal |

**Total Estimate:** ~$0.50/hour while running

### **After Deletion:**
- ✅ All charges stopped
- ✅ No lingering resources
- ✅ Clean subscription state

---

## **Important Notes**

### **⚠️ Run From Local Machine**

Do NOT run deletion commands from the jumpbox:
1. Jumpbox is inside the resource group being deleted
2. SSH connection will drop mid-deletion
3. Could leave resources in inconsistent state

### **⚠️ Deletion is Permanent**

- No undo for resource group deletion
- All data (certificates, configs, etc.) on VMs is lost
- Ensure you've saved anything needed locally first

### **💾 What to Save Before Deletion**

If you want to preserve anything:
```bash
# From jumpbox before deletion
scp root@jumpbox:~/ca.* ./backup/
scp root@jumpbox:~/machines.txt ./backup/
scp root@jumpbox:~/*.crt ./backup/
scp root@jumpbox:~/*.key ./backup/
```

---

## **Troubleshooting**

### **Issue: "AuthorizationFailed"**

**Cause:** Insufficient permissions

**Fix:** Ensure you're logged in with the correct Azure account:
```bash
az account show
az login  # If needed
```

---

### **Issue: Deletion Taking Too Long**

**Cause:** Azure is cleaning up resources in order

**Check:**
```bash
az group show --name kthw-learning --query "properties.provisioningState"
```

**Normal:** Can take up to 15 minutes for complex resource groups

---

### **Issue: Some Resources Remain**

**Cause:** Resource locks or dependencies

**Fix:**
```bash
# Check for locks
az lock list --resource-group kthw-learning

# Delete locks if any
az lock delete --name <lock-name> --resource-group kthw-learning

# Retry deletion
az group delete --name kthw-learning --yes
```

---

## **Section 13 Accomplishments**

- ✅ Verified resource group name
- ✅ Deleted all Azure resources
- ✅ Confirmed deletion complete
- ✅ Charges stopped

---

## **Key Takeaways**

1. **Resource groups enable clean teardown** - Delete one thing, everything goes

2. **Always verify before deleting** - Check you're targeting the right resource group

3. **Run from local machine** - Don't delete infrastructure you're connected through

4. **`--no-wait` for background deletion** - Returns immediately while Azure cleans up

5. **Monitor until "not found"** - That's your confirmation charges have stopped

---

## **KTHW on Azure - Complete! 🎉**

### **What I Built:**
- Kubernetes cluster from scratch on Azure
- 4 VMs: jumpbox, control plane, 2 workers
- Full TLS/PKI infrastructure
- etcd, API server, controller manager, scheduler
- kubelet, kube-proxy, containerd on workers
- Pod networking with manual routes
- All smoke tests passing

### **What I Learned:**
- Azure resource management
- Kubernetes internals (not just `kubectl apply`)
- TLS certificate generation and debugging
- RBAC configuration
- Linux networking and routing
- Troubleshooting real infrastructure issues

---

## **Commands Reference**

```bash
# Verify resource group
az group list --output table

# Delete everything
az group delete --name kthw-learning --yes --no-wait

# Monitor deletion
az group show --name kthw-learning

# Confirm deletion complete (should error)
az group list --output table | grep kthw
```
