# Section 6: Data Encryption Config and Key

## Overview

This section generates an encryption configuration that tells Kubernetes how to encrypt sensitive data (Secrets) at rest in etcd using AES-256 encryption.

**Safe Deposit Box Analogy:**
- **etcd** = Bank vault (stores all cluster data)
- **Secrets** = Valuable items you want to protect
- **Without encryption** = Items stored in transparent bags (base64) - anyone with vault access can see contents
- **With encryption** = Items in locked safe deposit boxes - vault access alone isn't enough, you need the key
- **Encryption key** = The physical key to your safe deposit box
- **Encryption config** = Bank's policy on which boxes to use and how to lock them

---

## Why Encryption at Rest?

**The Problem:**

By default, Kubernetes Secrets are only base64-encoded (NOT encrypted) in etcd:

```bash
# Secret value: "super-secret-password"
# Stored in etcd as: "c3VwZXItc2VjcmV0LXBhc3N3b3Jk"

# Anyone with etcd access can decode:
echo "c3VwZXItc2VjcmV0LXBhc3N3b3Jk" | base64 -d
# Output: super-secret-password  (readable!)
```

**The Solution:**

With encryption at rest enabled, the same secret is stored as encrypted binary data that's unreadable without the encryption key.

- **What gets encrypted:** Kubernetes Secrets (passwords, API tokens, TLS certificates stored as secrets)
- **What stays unencrypted:** Other resources (ConfigMaps, Pods, Services) - they don't contain sensitive data

---

## How Encryption at Rest Works

```text
┌─────────────────────────────────────────────────────────────┐
│  1. kubectl create secret (from user)                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  2. API Server receives secret                             │
│     Reads encryption-config.yaml                            │
│     Finds encryption key                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  3. API Server encrypts secret data                        │
│     Uses AES-CBC with 256-bit key                          │
│     Prepends: k8s:enc:aescbc:v1:key1:                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Encrypted data stored in etcd                          │
│     k8s:enc:aescbc:v1:key1:��Ϭ�8�... (binary)             │
└─────────────────────┬───────────────────────────────────────┘
```

Reading process is reversed:

- API server reads from etcd
- Detects `k8s:enc:aescbc:v1:key1:` prefix
- Decrypts using the key from encryption-config.yaml
- Returns plaintext to authorized user

---

## Encryption Config Structure

The encryption config file tells the API server:

- **What to encrypt:** `resources: - secrets` (which Kubernetes resources)
- **How to encrypt:** `aescbc` (AES-CBC algorithm)
- **Encryption key:** Base64-encoded 32-byte random key
- **Fallback:** `identity: {}` (can still read old unencrypted data during migration)

File structure:

```yaml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

Provider order matters:

- `aescbc` (first) = Used for new writes (encrypt new secrets)
- `identity` (second) = Fallback for reads (can read old unencrypted data)

---

## Step 1: Generate Encryption Key

**Purpose:** Create a cryptographically random 32-byte (256-bit) encryption key.

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

What this does:

- `head -c 32 /dev/urandom` - Read 32 random bytes from Linux random number generator
- `base64` - Encode as base64 (YAML-safe format)
- `export ENCRYPTION_KEY=...` - Store in environment variable for next step

**Key strength:** 32 bytes = 256 bits (same as AES-256, cryptographically strong)

**Why /dev/urandom:** Provides cryptographically secure random numbers from kernel entropy pool

---

## Step 2: Generate Encryption Config File

**Purpose:** Create encryption-config.yaml with the generated key embedded.

```bash
envsubst < configs/encryption-config.yaml > encryption-config.yaml
```

What this does:

- `envsubst` - Substitutes environment variables in template
- `configs/encryption-config.yaml` - Template file (contains `${ENCRYPTION_KEY}` placeholder)
- `> encryption-config.yaml` - Output final config file

Template file (`configs/encryption-config.yaml`):

```yaml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
```

Output file (`encryption-config.yaml`):

```yaml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: bXlzdXBlcnNlY3JldGVuY3J5cHRpb25rZXk=  # Actual key
      - identity: {}
```

---

## Step 3: Distribute Encryption Config to Server

**Purpose:** Copy encryption config to server node where API server will use it.

```bash
scp encryption-config.yaml root@server:~/
```

**Why only server:** Only the API server needs this config - it's the component that encrypts/decrypts data before storing in etcd.

**Where it goes:** `/root/encryption-config.yaml` on server (API server will load it during bootstrap in Section 8)

---

## Verification

Check file was created:

```bash
ls -lh encryption-config.yaml
```

Expected: File exists (~271 bytes)

Check file was copied to server:

```bash
ssh root@server "ls -lh ~/encryption-config.yaml"
```

Expected output:

```text
-rw------- 1 root root 271 Jan  2 20:XX /root/encryption-config.yaml
```

Inspect the encryption key (optional):

```bash
cat encryption-config.yaml
```

You should see:

- `kind: EncryptionConfig`
- `resources: - secrets`
- `aescbc:` with a long base64 key
- `identity: {}` fallback

---

## Security Considerations

### Encryption Key Protection

**Critical:** The encryption key is as sensitive as all your secrets combined!

If key is compromised:

- ❌ Attacker with etcd access can decrypt ALL secrets
- ❌ Entire encryption is pointless

Protection in tutorial:

- Key stored in `/root/encryption-config.yaml` (600 permissions)
- Only accessible by root on server node

Protection in production:

- Use KMS providers (AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault)
- Key stored in hardware security module (HSM)
- Key rotation policy (rotate keys regularly)
- Envelope encryption (encrypt the encryption key itself)

---

## Production Differences

| Aspect | Tutorial | Production |
|---|---|---|
| Provider | `aescbc` (static key) | `kms` (external key management) |
| Key storage | Filesystem | HSM, cloud KMS, Vault |
| Key rotation | Manual | Automated (every 90 days) |
| Audit | None | Key access logging |
| Backup | File backup | KMS handles automatically |

---

## Encryption Providers

Kubernetes supports multiple encryption providers (in order of strength):

| Provider | Description | Use Case |
|---|---|---|
| `kms` | External KMS (AWS/GCP/Azure) | Production (best) |
| `aesgcm` | AES-GCM (authenticated encryption) | Strong alternative to KMS |
| `aescbc` | AES-CBC (what you use) | Tutorial/basic production |
| `secretbox` | XSalsa20-Poly1305 | Alternative strong encryption |
| `identity` | No encryption (passthrough) | Fallback for migration |

**Why you use `aescbc`:** Simple, no external dependencies, good enough for learning. Production should use KMS.

---

## What Happens Next

- Section 7 (etcd bootstrap): etcd will be installed and started (data store)
- Section 8 (API server bootstrap): API server will be configured with `--encryption-provider-config=/root/encryption-config.yaml` flag - this activates encryption

When encryption activates:

- New secrets → encrypted with `aescbc` provider
- Old secrets (if any) → still readable via `identity` provider
- All secrets in etcd → encrypted at rest

---

## Testing Encryption (After Section 8)

Once API server is running, you can verify encryption:

```bash
# Create a test secret
kubectl create secret generic test-secret --from-literal=password=my-password

# Read from etcd directly (on server)
ETCDCTL_API=3 etcdctl get /registry/secrets/default/test-secret   --endpoints=https://127.0.0.1:2379   --cacert=/etc/etcd/ca.crt   --cert=/etc/etcd/etcd-server.crt   --key=/etc/etcd/etcd-server.key

# Output should start with: k8s:enc:aescbc:v1:key1:
# Followed by encrypted binary data (unreadable)
```

Without encryption it would show: base64-encoded plaintext

---

## Files Created in Section 6

### On jumpbox (`~/kubernetes-the-hard-way/`)

- `encryption-config.yaml` (271 bytes) - Contains encryption key and config

### On server

- `~/encryption-config.yaml` (271 bytes) - Will be used by API server

**Total:** 1 config file (replicated to server)
