# Section 4: Certificate Authority & PKI Setup

## Overview

Kubernetes uses **zero-trust security** where every component must cryptographically prove its identity. This is achieved through **Public Key Infrastructure (PKI)** using **TLS certificates**.

**Passport System Analogy:**
- **Government** = Certificate Authority (CA), which issues and validates all passports
- **Passport** = Certificate (`.crt` file), your public proof of identity
- **Fingerprint** = Private Key (`.key` file), a secret signature only you possess
- **Border Control** = TLS verification, which validates passports against government records

Just as border control trusts your passport because it's government-issued, Kubernetes components trust each other's certificates because they're **CA-signed**.

---

## Architecture Diagram

```text
                ┌─────────────────────┐
                │  CA (Root of Trust) │
                │   ca.key + ca.crt   │
                └──────────┬──────────┘
                           │ Signs all certificates
      ┌────────────────────┼────────────────────┐
      ▼                    ▼                    ▼
┌──────────┐         ┌──────────┐        ┌──────────┐
│ Jumpbox  │         │  Server  │        │ Workers  │
│          │         │          │        │          │
│ admin    │────────>│ kube-api │<───────│ kubelet  │
│ .crt/key │         │ .crt/key │        │ .crt/key │
└──────────┘         │          │        │          │
                     │ ca.key   │        │ ca.crt   │
                     │ (signs   │        │ (verify) │
                     │  tokens) │        └──────────┘
                     └──────────┘

     All components authenticate via mTLS
```

---

## Core Concepts

### Certificate Authority (CA)
**What:** Root of trust that issues and signs all certificates  
**Why:** Establishes trust hierarchy; all components trust the same authority  
**Analogy:** Government passport office  

**Components:**
- `ca.crt`: Public certificate (distributed to all VMs)
- `ca.key`: Private key (**SECRET**, only on server)

### Private Key
**What:** Secret cryptographic signature ability  
**Why:** Proves identity by signing data only the real owner could sign  
**Analogy:** Your unique fingerprint  
**Security:** Never shared, stays on owning component

### Certificate
**What:** Public proof of identity signed by CA  
**Why:** Allows identity verification without private key  
**Analogy:** Passport with government signature  

**Contains:**
- **CN (Common Name):** Identity for RBAC (e.g., `admin`)
- **O (Organization):** Group membership (e.g., `system:masters`)
- **Validity Period:** How long certificate is valid

### Subject Alternative Names (SANs)
**What:** Multiple valid addresses/names for the same identity  
**Why:** API server accessible via multiple IPs/DNS names  
**Analogy:** Passport valid at any border crossing  

**Example:** `kubernetes`, `10.32.0.1`, `10.240.0.10`, `127.0.0.1`

---

## Step 1: Create CA Configuration File

**Purpose:** Define all certificate specifications in one file instead of typing parameters repeatedly.

```bash
cd ~/kubernetes-the-hard-way

cat > ca.conf << 'EOF'
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
C = UK
ST = England
L = London
O = Kubernetes
OU = KTHW
CN = Kubernetes

[v3_ca]
basicConstraints = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash

[admin]
distinguished_name = dn_admin
req_extensions = v3_admin
prompt = no

[dn_admin]
CN = admin
O = system:masters

[v3_admin]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectKeyIdentifier = hash

[node-0]
distinguished_name = dn_node-0
req_extensions = v3_node-0
prompt = no

[dn_node-0]
CN = system:node:node-0
O = system:nodes

[v3_node-0]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = DNS:node-0.kubernetes.local,IP:10.240.0.20
subjectKeyIdentifier = hash

[node-1]
distinguished_name = dn_node-1
req_extensions = v3_node-1
prompt = no

[dn_node-1]
CN = system:node:node-1
O = system:nodes

[v3_node-1]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = DNS:node-1.kubernetes.local,IP:10.240.0.21
subjectKeyIdentifier = hash

[kube-proxy]
distinguished_name = dn_kube-proxy
req_extensions = v3_kube-proxy
prompt = no

[dn_kube-proxy]
CN = system:kube-proxy

[v3_kube-proxy]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectKeyIdentifier = hash

[kube-scheduler]
distinguished_name = dn_kube-scheduler
req_extensions = v3_kube-scheduler
prompt = no

[dn_kube-scheduler]
CN = system:kube-scheduler

[v3_kube-scheduler]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectKeyIdentifier = hash

[kube-controller-manager]
distinguished_name = dn_kube-controller-manager
req_extensions = v3_kube-controller-manager
prompt = no

[dn_kube-controller-manager]
CN = system:kube-controller-manager

[v3_kube-controller-manager]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectKeyIdentifier = hash

[kube-apiserver]
distinguished_name = dn_kube-apiserver
req_extensions = v3_kube-apiserver
prompt = no

[dn_kube-apiserver]
CN = kube-apiserver

[v3_kube-apiserver]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names_kube-apiserver
subjectKeyIdentifier = hash

[alt_names_kube-apiserver]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = server.kubernetes.local
IP.1 = 10.32.0.1
IP.2 = 10.240.0.10
IP.3 = 127.0.0.1

[service-accounts]
distinguished_name = dn_service-accounts
req_extensions = v3_service-accounts
prompt = no

[dn_service-accounts]
CN = service-accounts

[v3_service-accounts]
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectKeyIdentifier = hash
EOF
```

---

## Step 2: Generate Certificate Authority

**Purpose:** Create root CA that signs all component certificates.

### Generate 4096-bit RSA private key
```bash
openssl genrsa -out ca.key 4096
```

### Generate self-signed CA certificate valid 10 years
```bash
openssl req -x509 -new -sha512 -noenc   -key ca.key -days 3653   -config ca.conf   -out ca.crt
```

### Verify CA certificate
```bash
openssl x509 -in ca.crt -text -noout
```

---

## Step 3: Generate Component Certificates

**Purpose:** Create certificates for Kubernetes components.

```bash
for i in admin node-0 node-1 kube-proxy kube-scheduler kube-controller-manager kube-apiserver service-accounts; do
  echo "Generating certificate for: ${i}"

  openssl genrsa -out ${i}.key 4096

  openssl req -new -key ${i}.key -sha256     -config ca.conf -section ${i}     -out ${i}.csr

  openssl x509 -req -days 3653 -in ${i}.csr     -copy_extensions copyall     -sha512 -CA ca.crt -CAkey ca.key     -CAcreateserial -out ${i}.crt
done
```

### What this creates
- 8 private keys (`.key` files), 4096-bit RSA
- 8 certificate signing requests (`.csr` files)
- 8 signed certificates (`.crt` files)

### Verify files created
```bash
ls -lh *.crt *.key
```

**Expected:** 9 `.crt` files, 9 `.key` files (CA + 8 components)

---

## Step 4: Distribute Certificates to VMs

**Purpose:** Copy certificates to VMs using the principle of least privilege. Each VM only gets what it needs.

### Worker node-0
```bash
scp ca.crt node-0.crt node-0.key root@10.240.0.20:~/
```

### Worker node-1
```bash
scp ca.crt node-1.crt node-1.key root@10.240.0.21:~/
```

### Server node
```bash
scp ca.crt ca.key kube-apiserver.crt kube-apiserver.key   service-accounts.crt service-accounts.key root@10.240.0.10:~/
```

### Distribution logic

| VM | Files | Why |
|---|---|---|
| node-0 | `ca.crt`, `node-0.crt`, `node-0.key` | Kubelet proves identity + verifies API server |
| node-1 | `ca.crt`, `node-1.crt`, `node-1.key` | Kubelet proves identity + verifies API server |
| server | `ca.crt`, `ca.key`, `kube-apiserver.crt/key`, `service-accounts.crt/key` | API server identity + signs pod tokens |
| jumpbox | (All files stay here) | Will create kubeconfig files in Section 5 |

---

## Step 5: Verify Distribution

### Verify node-0
```bash
ssh root@10.240.0.20 "ls -lh ~/*.crt ~/*.key"
```
**Expected:** 3 files (`ca.crt`, `node-0.crt`, `node-0.key`)

### Verify node-1
```bash
ssh root@10.240.0.21 "ls -lh ~/*.crt ~/*.key"
```
**Expected:** 3 files (`ca.crt`, `node-1.crt`, `node-1.key`)

### Verify server
```bash
ssh root@10.240.0.10 "ls -lh ~/*.crt ~/*.key"
```
**Expected:** 6 files (`ca.crt`, `ca.key`, `kube-apiserver.crt/key`, `service-accounts.crt/key`)

---

## How mTLS Authentication Works

1. Kubelet → API Server: "I want to connect"
2. API Server → Kubelet: `kube-apiserver.crt` (proves "I am API server")
3. Kubelet verifies: check signature using `ca.crt` ✓
4. Kubelet → API Server: `node-0.crt` + signature from `node-0.key`
5. API Server verifies: check signature using `ca.crt` ✓
6. API Server extracts: `CN=system:node:node-0`, `O=system:nodes`
7. RBAC checks: Is `system:node:node-0` allowed? Yes ✓
8. Encrypted TLS channel established ✓

---

## Certificate Summary

| Certificate | Location | CN | O | Purpose |
|------------|----------|----|---|---------|
| CA | Server (key), All VMs (cert) | Kubernetes | Kubernetes | Root of trust |
| admin | Jumpbox | admin | system:masters | `kubectl` commands |
| node-0 | node-0 VM | system:node:node-0 | system:nodes | Kubelet identity |
| node-1 | node-1 VM | system:node:node-1 | system:nodes | Kubelet identity |
| kube-proxy | Jumpbox | system:kube-proxy | - | Network proxy |
| kube-scheduler | Jumpbox | system:kube-scheduler | - | Pod scheduler |
| kube-controller-manager | Jumpbox | system:kube-controller-manager | - | Controllers |
| kube-apiserver | Server | kube-apiserver | - | API server |
| service-accounts | Server | service-accounts | - | Pod token signing |

---

## Security Notes

### Certificate Validity
- **Tutorial:** 10 years (convenience for learning)
- **Production:** 90-365 days (10 years is too long!)
- **Best Practice:** Automated rotation with `cert-manager`

### CA Private Key Protection
- **Tutorial:** Server filesystem with `600` permissions
- **Production:** HSM, Vault, or cloud KMS (air-gapped from cluster)

### Private Key Rules
- ❌ NEVER copy between systems (except initial distribution)
- ❌ NEVER commit to version control
- ❌ NEVER share via insecure channels
- ✅ ALWAYS use `600` permissions
- ✅ ALWAYS use encrypted transfer (SCP/SSH)
