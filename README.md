# Kubernetes The Hard Way - On Azure

![Azure](https://img.shields.io/badge/Azure-Compatible-blue)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.32.3-brightgreen)
![License](https://img.shields.io/badge/License-MIT-yellow)

**Building production-grade Kubernetes from scratch to understand what managed services abstract away.**

I built a complete Kubernetes cluster manually on Azure (no AKS, no automation, no shortcuts) to understand how every component works, connects, and fails. This repository documents that journey, including the real problems I solved that don't appear in tutorials.

---

## Why I Built This

**The Problem:** Working with managed Kubernetes (AKS, EKS, GKE) gives you a cluster, but hides the internals. When things break in production, you're troubleshooting a black box; there's hardly anything you can get from the control plane as it's managed by the cloud provider.

**The Goal:** Understand Kubernetes from first principles: TLS certificates, etcd consensus, kubelet communication, pod networking. This lets me debug production issues with confidence.

**The Result:** Complete understanding of:
- How the control plane components coordinate (API server, scheduler, controller manager)
- Why TLS certificate configuration matters (learned this the hard way, literally)
- What kubelets actually do on worker nodes
- How pod networking routes traffic between nodes
- Where cluster state lives and why etcd is critical

---

## What I Learned (That You Won't Get From Managed K8s)

### 1. TLS is Everything
Hit a production-realistic issue: `kubectl logs` failed with certificate verification errors. Root cause? Node certificates only included FQDNs, not short hostnames.

**The fix:** Regenerated certificates with proper Subject Alternative Names (SANs). **The lesson:** Certificate misconfiguration causes silent failures that look like networking issues.

### 2. RBAC Applies to Internal Components
The API server itself needs RBAC permissions to access kubelets. This isn't documented in managed services because it's pre-configured. When building manually, you learn these dependencies exist.

### 3. Azure Quota Gotchas
New Azure accounts block `Standard_B2s` and `Standard_D2s_v5` VMs. Had to discover `Standard_D2s_v6` works through trial and `az vm list-skus`. Real-world platform engineering.

### 4. etcd is a Single Point of Failure (in this setup)
Tutorial uses single-node etcd. Production needs 3 or 5 for high availability. Understanding WHY (Raft consensus requires quorum) matters when designing resilient systems.

### 5. Pod Networking is Manual Without CNI
Had to configure static routes so pods on node-0 could reach pods on node-1. Production uses CNI plugins (Calico, Cilium) that automate this. Now I understand what those plugins actually DO.

---

## How This Differs From Managed Kubernetes

| What You Learn | Managed K8s (AKS) | This Project |
|----------------|-------------------|--------------|
| Certificate Management | Automated, invisible | Generated all certs manually, debugged SAN issues |
| Control Plane | Abstracted away | Installed API server, scheduler, controller manager by hand |
| etcd | Provider manages it | Bootstrapped etcd, understand what it stores |
| Worker Setup | Node pools auto-provision | Installed kubelet, containerd, kube-proxy manually |
| Networking | Azure CNI does it | Configured routes, understand IP allocation |
| Troubleshooting | "Submit a ticket" | Full control: can SSH to any node, check logs |

**The difference:** With managed K8s, you're a user. With this, you're an operator who understands the system.

---

## What Problems I Solved (Azure-Specific)

### Problem 1: VM Size Restrictions
**Issue:** New Azure accounts have quota limits. `Standard_B2s` rejected in multiple regions.

**Solution:** Used `az vm list-skus` to find available sizes. `Standard_D2s_v6` worked.

**Why it matters:** Production platforms face quota limits. Knowing how to query availability prevents blocked deployments.

### Problem 2: Certificate SAN Mismatch
**Issue:** `kubectl logs` failed: `x509: certificate is valid for node-1.kubernetes.local, not node-1`

**Solution:**
1. Checked node registration names vs certificate SANs
2. Updated `ca.conf` to include both FQDNs and short hostnames
3. Regenerated node certificates
4. Distributed to workers and restarted kubelets

**Why it matters:** Certificate issues look like networking failures. Systematic debugging (check registration → check certs → compare) is critical.

### Problem 3: RBAC Permissions for API Server
**Issue:** `kubectl port-forward` failed: `Forbidden (user=kube-apiserver, verb=create, resource=nodes, subresource=proxy)`

**Solution:** Created ClusterRoleBinding granting `kube-apiserver` user the `system:kubelet-api-admin` role.

**Why it matters:** Internal components need permissions too. Managed services pre-configure this; manual setup teaches you the dependencies.

---

## Who Should Use This

**You should follow this if you:**
- ✅ Work with Kubernetes but want to understand the internals
- ✅ Debug production K8s issues and feel like you're guessing
- ✅ Want to learn Azure while learning Kubernetes
- ✅ Prepare for senior platform engineering roles
- ✅ Need a portfolio project that demonstrates depth

**Skip this if you:**
- ❌ Just want a cluster (use AKS instead)
- ❌ Don't care how Kubernetes works internally
- ❌ Can't invest time; this takes time

**Cost:** Azure resources cost money. The longer you keep VMs running, the more you pay. Run Section 13 cleanup immediately when done. Use Azure's pay-as-you-go subscription (free trial has VM size limits).

---

## Sections

| Section | Description |
|---------|-------------|
| [00 - Initial Setup](docs/00-intro.md) | Azure account setup, CLI installation, resource group creation |
| [01 - Jumpbox](docs/01-jumpbox.md) | Bastion host setup, Kubernetes binaries |
| [02 - Compute Resources (Terminology)](docs/02-compute-resources-terminology.md) | Concepts and terminology breakdown |
| [03 - Compute Resources (Implementation)](docs/03-compute-resources-implementation.md) | VNet, VMs, SSH, hostname configuration |
| [04 - Certificate Authority](docs/04-certificate-authority.md) | PKI setup, TLS certificates |
| [05 - Kubernetes Configuration Files](docs/05-kubernetes-configuration-files.md) | kubeconfig files for components |
| [06 - Data Encryption Keys](docs/06-data-encryption-keys.md) | Encryption config for secrets at rest |
| [07 - Bootstrapping etcd](docs/07-bootstrapping-etcd.md) | etcd cluster setup |
| [08 - Bootstrapping Control Plane](docs/08-bootstrapping-control-plane.md) | API server, controller manager, scheduler |
| [09 - Bootstrapping Workers](docs/09-bootstrapping-workers.md) | kubelet, kube-proxy, containerd |
| [10 - Configuring kubectl](docs/10-configuring-kubectl.md) | Remote kubectl access |
| [11 - Pod Network Routes](docs/11-pod-network-routes.md) | Cross-node pod networking |
| [12 - Smoke Test](docs/12-smoke-test.md) | End-to-end cluster validation ← *Includes certificate debugging* |
| [13 - Tear Down](docs/13-tear-down.md) | Cleanup and resource deletion |

---

## Cluster Architecture

```
Jumpbox (10.240.0.4)
  ↓ SSH access via hostnames
  ├─ server (10.240.0.10) - Control plane
  ├─ node-0 (10.240.0.20) - Worker, Pod CIDR: 10.200.0.0/24
  └─ node-1 (10.240.0.21) - Worker, Pod CIDR: 10.200.1.0/24
```

---

## Key Differences from Original KTHW

| Aspect | Original KTHW | This Version |
|--------|---------------|--------------|
| Platform | Local VMs (Multipass) | Azure VMs |
| Networking | Host-only network | Azure VNet |
| Initial User | root | ubuntu (then enable root) |
| Cost | Free | Pay-as-you-go |
| Troubleshooting | Minimal | Extensive (certificate issues, RBAC, Azure quotas) |

---

## What You'll Build

By the end of this tutorial, you'll have:

- **Control Plane:** API server, controller manager, scheduler, etcd
- **Workers:** 2 nodes running kubelet, kube-proxy, containerd
- **Networking:** Pod CIDRs with manual routes, Service networking via kube-proxy
- **Security:** Complete PKI with mTLS between all components
- **Validation:** All smoke tests passing (encryption, deployments, services, logs, exec)

Most importantly: **You'll understand HOW it works, not just that it works.**

---

## Prerequisites

- Azure account with pay-as-you-go subscription (free trial has quota limits)
- Azure CLI installed locally
- Basic familiarity with Linux and SSH

---

## References

- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) - Kelsey Hightower's original tutorial. An invaluable resource that this project is based on.
- [Azure CLI Documentation](https://learn.microsoft.com/en-us/cli/azure/)

---

## License

MIT License - Feel free to use this for learning. If it helped you understand K8s better, star the repo! ⭐
