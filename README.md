# Kubernetes The Hard Way - On Azure

My documentation of building a Kubernetes cluster from scratch on Microsoft Azure, following Kelsey Hightower's [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) tutorial.

## What This Is

Step-by-step notes adapting KTHW for Azure instead of local VMs. Includes:

- Azure-specific commands and configurations
- Troubleshooting issues I encountered
- Explanations of concepts with analogies
- Production context and differences

## Sections

| Section | Description |
|---------|-------------|
| [00 - Initial Setup](intro-00.md) | Azure account setup, CLI installation, resource group creation |
| [01 - Jumpbox](jumpbox-01.md) | Bastion host setup, Kubernetes binaries |
| [02 - Compute Resources (Terminology)](compute-resources-02.md) | Concepts and terminology breakdown |
| [03 - Compute Resources (Implementation)](compute-resources-03.md) | VNet, VMs, SSH, hostname configuration |
| [04 - Certificate Authority](certificate-authority-04.md) | PKI setup, TLS certificates |
| [05 - Kubernetes Configuration Files](kubernetes-configuration-files-05.md) | kubeconfig files for components |
| [06 - Data Encryption Keys](data-encryption-keys-06.md) | Encryption config for secrets at rest |
| [07 - Bootstrapping etcd](bootstrapping-etcd-07.md) | etcd cluster setup |
| [08 - Bootstrapping Control Plane](bootstrapping-kubernetes-controllers-08.md) | API server, controller manager, scheduler |
| [09 - Bootstrapping Workers](bootstrapping-kubernetes-workers-09.md) | kubelet, kube-proxy, containerd |
| [10 - Configuring kubectl](configuring-kubectl-for-remote-access-10.md) | Remote kubectl access |
| [11 - Pod Network Routes](provisioning-pod-network-routes-11.md) | Cross-node pod networking |
| [12 - Smoke Test](smoke-test-12.md) | End-to-end cluster validation |
| [13 - Tear Down](tear-down-infra-13.md) | Cleanup and resource deletion |

## Cluster Architecture

```
Jumpbox (10.240.0.4)
  ↓ SSH access via hostnames
  ├─ server (10.240.0.10) - Control plane
  ├─ node-0 (10.240.0.20) - Worker, Pod CIDR: 10.200.0.0/24
  └─ node-1 (10.240.0.21) - Worker, Pod CIDR: 10.200.1.0/24
```

## Key Differences from Original KTHW

| Aspect | Original KTHW | This Version |
|--------|---------------|--------------|
| Platform | Local VMs (Multipass) | Azure VMs |
| Networking | Host-only network | Azure VNet |
| Initial User | root | ubuntu (then enable root) |
| Cost | Free | Pay-as-you-go |

## Prerequisites

- Azure account with pay-as-you-go subscription (free trial has quota limits)
- Azure CLI installed locally
- Basic familiarity with Linux and SSH

## References

- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) - Original tutorial by Kelsey Hightower
- [Azure CLI Documentation](https://learn.microsoft.com/en-us/cli/azure/)
