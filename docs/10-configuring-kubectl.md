# KTHW on Azure - Section 10: Configuring kubectl for Remote Access

**KTHW Tutorial:** https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/10-configuring-kubectl.md

---

## **Section Overview**

**Goal:** Configure kubectl on the jumpbox to manage the cluster without specifying config files each time.

**What I Achieved:**
```
Before: kubectl --kubeconfig admin.kubeconfig get nodes
After:  kubectl get nodes
```

**Capabilities After This Section:**
1. ✅ kubectl uses default config (~/.kube/config)
2. ✅ No manual --kubeconfig flag needed
3. ✅ Cluster, credentials, and context permanently configured
4. ✅ Ready for day-to-day cluster management

---

## **Why This Section Matters**

### **The Problem**

Every kubectl command previously required:
```bash
kubectl --kubeconfig admin.kubeconfig get pods
kubectl --kubeconfig admin.kubeconfig get nodes
kubectl --kubeconfig admin.kubeconfig describe deployment nginx
```

**Pain Points:**
- Repetitive typing
- Easy to forget the flag
- Error-prone in scripts
- Not how production kubectl usage works

### **The Solution**

Configure kubectl's default config file (`~/.kube/config`) with:
1. **Cluster** - Where the API server is and how to trust it
2. **Credentials** - How to authenticate (admin certs)
3. **Context** - Links cluster + credentials together

---

## **PART 1: Verify API Server Connectivity**

### **Step 1.1: Test HTTPS Connection**

**Command:**
```bash
curl --cacert ca.crt https://server.kubernetes.local:6443/version
```

**What It Does:**
- Makes an HTTPS request to the API server
- Uses the CA certificate to verify the server's identity
- Hits the `/version` endpoint to confirm API server is responding

**Analogy:**
Like calling a secure phone number and verifying it's really the right company before sharing information.

**Technical Details:**
- `--cacert ca.crt` = Trust certificates signed by this CA
- `https://server.kubernetes.local:6443` = API server address
- `/version` = Unauthenticated endpoint (returns K8s version info)

**Expected Output:**
```json
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  ...
}
```

**Why This Step:**
Confirms network connectivity and TLS before configuring kubectl.

---

## **PART 2: Configure kubectl**

### **The Three Components**

kubectl configuration has three parts that work together:

```
┌─────────────────────────────────────────────────────────┐
│                    ~/.kube/config                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  CLUSTER                 USER                           │
│  ├─ name: kubernetes     ├─ name: admin                 │
│  ├─ server: https://...  ├─ client-certificate          │
│  └─ ca-certificate       └─ client-key                  │
│           │                       │                     │
│           └───────┬───────────────┘                     │
│                   ▼                                     │
│              CONTEXT                                    │
│              ├─ name: kubernetes-the-hard-way           │
│              ├─ cluster: kubernetes-the-hard-way        │
│              └─ user: admin                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### **Step 2.1: Set Cluster Configuration**

**Command:**
```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443
```

**What It Does:**
Registers cluster connection details - where it is and how to trust it.

**Analogy:**
Saving a contact in your phone with name, number, and photo verification.

**Parameter Breakdown:**

**`set-cluster kubernetes-the-hard-way`**
- Creates/updates a cluster entry with this name
- Name is arbitrary but should be descriptive

**`--certificate-authority=ca.crt`**
- CA certificate to verify API server's TLS cert
- Ensures you're talking to the real API server

**`--embed-certs=true`**
- Base64-encodes the CA cert into the config file
- Makes config portable (no external file dependencies)

**`--server=https://server.kubernetes.local:6443`**
- API server URL
- Uses FQDN I configured in /etc/hosts

**Technical Details:**
Creates cluster entry in `~/.kube/config` with:
- Server URL for API connections
- Embedded base64-encoded CA cert for TLS verification

---

### **Step 2.2: Set Admin Credentials**

**Command:**
```bash
kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key
```

**What It Does:**
Registers your authentication credentials (certificate and private key).

**Analogy:**
Adding your passport and ID to your wallet for identity verification.

**Parameter Breakdown:**

**`set-credentials admin`**
- Creates/updates a user entry named "admin"
- Name must match what's used in the context

**`--client-certificate=admin.crt`**
- Your identity certificate (signed by cluster CA)
- API server trusts certs signed by the CA

**`--client-key=admin.key`**
- Private key for the certificate
- Proves you own the certificate

**Technical Details:**
Stores admin client certificate and private key in kubeconfig for mTLS (mutual TLS) authentication to API server.

**How mTLS Works:**
```
1. kubectl connects to API server
2. API server presents its certificate (I verify with CA)
3. kubectl presents admin.crt (API server verifies with CA)
4. Both sides trust each other → connection established
```

---

### **Step 2.3: Create Context**

**Command:**
```bash
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

**What It Does:**
Links a cluster with credentials - creates a usable connection profile.

**Analogy:**
Creating a speed dial that combines a contact (cluster) with your caller ID settings (credentials).

**Parameter Breakdown:**

**`set-context kubernetes-the-hard-way`**
- Creates/updates a context with this name
- Contexts are what you switch between

**`--cluster=kubernetes-the-hard-way`**
- References the cluster I created in Step 2.1
- Must match the cluster name exactly

**`--user=admin`**
- References the credentials I created in Step 2.2
- Must match the user name exactly

**Technical Details:**
Creates context binding cluster definition to user credentials - a named configuration kubectl can activate.

**Why Contexts Exist:**
In production, you might have:
```
contexts:
  - name: dev-cluster
    cluster: dev
    user: dev-admin
  - name: staging-cluster
    cluster: staging
    user: staging-admin
  - name: prod-cluster
    cluster: prod
    user: prod-readonly
```

Switch with: `kubectl config use-context prod-cluster`

---

### **Step 2.4: Activate Context**

**Command:**
```bash
kubectl config use-context kubernetes-the-hard-way
```

**What It Does:**
Sets this context as the default for all kubectl commands.

**Analogy:**
Setting this as your default phone line for all calls.

**Technical Details:**
Updates `current-context` field in `~/.kube/config` so kubectl uses these cluster/credentials by default.

**Before:**
```yaml
current-context: ""
```

**After:**
```yaml
current-context: kubernetes-the-hard-way
```

---

## **PART 3: Verification**

### **Step 3.1: Check Version**

**Command:**
```bash
kubectl version
```

**What It Does:**
Shows kubectl client version and server (API server) version.

**Expected Output:**
```
Client Version: v1.32.3
Kustomize Version: v5.5.0
Server Version: v1.32.3
```

**What This Proves:**
- ✅ kubectl can reach API server (Server Version appears)
- ✅ TLS verification working (no certificate errors)
- ✅ Authentication working (no 401/403 errors)
- ✅ Versions compatible (both v1.32.3)

---

### **Step 3.2: List Nodes**

**Command:**
```bash
kubectl get nodes
```

**What It Does:**
Queries API server for all registered nodes in the cluster.

**Expected Output:**
```
NAME     STATUS   ROLES    AGE   VERSION
node-0   Ready    <none>   13d   v1.32.3
node-1   Ready    <none>   13d   v1.32.3
```

**What This Proves:**
- ✅ Both worker nodes registered with API server
- ✅ Both nodes in Ready state (kubelet healthy)
- ✅ kubectl configured correctly for cluster access

---

## **What Changed**

### **Before This Section:**
```bash
# Every command needed explicit config path
kubectl --kubeconfig admin.kubeconfig get nodes
kubectl --kubeconfig admin.kubeconfig get pods
kubectl --kubeconfig admin.kubeconfig describe service nginx
```

### **After This Section:**
```bash
# kubectl uses ~/.kube/config automatically
kubectl get nodes
kubectl get pods
kubectl describe service nginx
```

### **Files Created:**
```
~/.kube/config    # kubectl's default configuration file
```

---

## **Understanding ~/.kube/config**

### **View Current Config:**
```bash
kubectl config view
```

### **Config Structure:**
```yaml
apiVersion: v1
kind: Config
current-context: kubernetes-the-hard-way

clusters:
  - name: kubernetes-the-hard-way
    cluster:
      certificate-authority-data: <base64-encoded-ca-cert>
      server: https://server.kubernetes.local:6443

users:
  - name: admin
    user:
      client-certificate-data: <base64-encoded-admin-cert>
      client-key-data: <base64-encoded-admin-key>

contexts:
  - name: kubernetes-the-hard-way
    context:
      cluster: kubernetes-the-hard-way
      user: admin
```

---

## **Production Differences**

### **Multiple Clusters**

In production, you typically have multiple clusters:

```yaml
contexts:
  - name: dev
    context:
      cluster: dev-cluster
      user: dev-admin
  - name: staging
    context:
      cluster: staging-cluster
      user: staging-admin
  - name: prod
    context:
      cluster: prod-cluster
      user: prod-readonly
```

**Switch Between:**
```bash
kubectl config use-context dev
kubectl config use-context prod
```

### **View Current Context:**
```bash
kubectl config current-context
```

### **List All Contexts:**
```bash
kubectl config get-contexts
```

---

## **Common kubectl config Commands**

```bash
# View full config
kubectl config view

# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <name>

# Set default namespace for current context
kubectl config set-context --current --namespace=<namespace>

# Delete a context
kubectl config delete-context <name>

# Delete a cluster
kubectl config delete-cluster <name>

# Delete credentials
kubectl config delete-user <name>
```

---

## **Troubleshooting**

### **Issue: "Unable to connect to the server"**

**Cause:** API server not reachable

**Check:**
```bash
ping server.kubernetes.local
curl --cacert ca.crt https://server.kubernetes.local:6443/version
```

**Fix:** Verify /etc/hosts entries and API server is running

---

### **Issue: "x509: certificate signed by unknown authority"**

**Cause:** CA certificate not configured or wrong CA

**Check:**
```bash
kubectl config view | grep certificate-authority
```

**Fix:** Re-run set-cluster with correct --certificate-authority

---

### **Issue: "error: You must be logged in to the server (Unauthorized)"**

**Cause:** Client certificate not recognized

**Check:**
```bash
kubectl config view | grep client-certificate
```

**Fix:** Verify admin.crt was signed by cluster CA, re-run set-credentials

---

### **Issue: "The connection to the server localhost:8080 was refused"**

**Cause:** No kubeconfig or context not set

**Check:**
```bash
kubectl config current-context
ls ~/.kube/config
```

**Fix:** Run use-context command or check config file exists

---

## **Section 10 Accomplishments**

### **Configuration:**
- ✅ Cluster registered in kubectl config
- ✅ Admin credentials stored
- ✅ Context created and activated
- ✅ Default config file populated (~/.kube/config)

### **Verification:**
- ✅ kubectl version shows client and server
- ✅ kubectl get nodes returns Ready workers
- ✅ No --kubeconfig flag needed

### **Usability:**
- ✅ kubectl commands work without extra flags
- ✅ Ready for normal cluster operations

---

## **Key Takeaways**

1. **kubectl config is three parts** - Cluster (where), User (who), Context (links them)

2. **Embed certs for portability** - `--embed-certs=true` makes config self-contained

3. **Contexts enable multi-cluster** - Switch between dev/staging/prod easily

4. **~/.kube/config is the default** - kubectl looks here unless --kubeconfig specified

5. **mTLS provides mutual trust** - Both client and server verify each other's certificates

---

## **Next: Section 11 - Provisioning Pod Network Routes**

Now that kubectl is configured, I can:
1. Configure pod-to-pod networking across nodes
2. Set up routes so pods on node-0 can reach pods on node-1
3. Enable cross-node pod communication

---

## **Commands Reference**

```bash
# Test API server
curl --cacert ca.crt https://server.kubernetes.local:6443/version

# Configure kubectl
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443

kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key

kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

kubectl config use-context kubernetes-the-hard-way

# Verify
kubectl version
kubectl get nodes
```
