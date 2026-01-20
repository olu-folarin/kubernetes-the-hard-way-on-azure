# KTHW on Azure - Section 12: Smoke Test

**KTHW Tutorial:** https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/12-smoke-test.md

---

## Table of Contents
- [Section Overview](#section-overview)
- [Test 1: Data Encryption at Rest](#test-1-data-encryption-at-rest)
- [Test 2: Deployments](#test-2-deployments)
- [Test 3: Port Forwarding](#test-3-port-forwarding)
- [Test 4: Container Logs](#test-4-container-logs)
- [Test 5: Execute Commands in Container](#test-5-execute-commands-in-container)
- [Test 6: Services (NodePort)](#test-6-services-nodeport)
- [Issues Encountered and Resolution](#issues-encountered-and-resolution)
- [Final Test Results](#final-test-results)
- [What I Learned](#what-i-learned)
- [Production Differences](#production-differences)
- [Troubleshooting](#troubleshooting)
- [Commands Reference](#commands-reference)

---

## **Section Overview**

**Goal:** Comprehensive end-to-end tests to verify the entire Kubernetes cluster is functioning correctly - from encryption to networking to container execution.

**Tests Performed:**
| Test | Component Validated | Result |
|------|---------------------|--------|
| 1. Data Encryption | etcd encryption at rest | ✅ PASSED |
| 2. Deployments | Controller Manager, Scheduler | ✅ PASSED |
| 3. Port Forwarding | kubectl → API → kubelet | ✅ PASSED* |
| 4. Container Logs | kubectl logs → kubelet | ✅ PASSED* |
| 5. Exec into Container | kubectl exec → kubelet | ✅ PASSED* |
| 6. Services (NodePort) | kube-proxy, networking | ✅ PASSED |

*Required certificate fix (documented below)

---

## **Test 1: Data Encryption at Rest**

### **Step 1.1: Create a Secret**

**Command:**
```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

**What It Does:**
Creates a Kubernetes Secret containing sensitive data.

**Analogy:**
Putting a confidential document in a locked safe.

**Technical Details:**
Sends secret data to API server, which encrypts it with the AES-CBC key from Section 6 before storing in etcd.

---

### **Step 1.2: Verify Encryption in etcd**

**Command:**
```bash
ssh root@server \
  'etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C'
```

**What It Does:**
Reads the raw encrypted data directly from etcd's database.

**Analogy:**
Opening the safe's internal mechanism to verify the document is actually locked, not just sitting there unprotected.

**Technical Details:**
- `etcdctl get` → Reads raw bytes from etcd's key-value store
- `hexdump -C` → Displays data in hexadecimal format with ASCII

**Expected Output:**
```
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 1e 9f 2a 2d 9c 32 3a  |:v1:key1:..*-.2:|
```

**Key Indicator:**
`k8s:enc:aescbc:v1:key1` proves the secret is encrypted with AES-CBC using key1 (not stored in plaintext).

**Result:** ✅ PASSED - Secret encrypted at rest

---

## **Test 2: Deployments**

### **Step 2.1: Create Deployment**

**Command:**
```bash
kubectl create deployment nginx --image=nginx:latest
```

**What It Does:**
Creates a Deployment that manages an nginx pod.

**Analogy:**
Hiring a manager (Deployment) to ensure at least one employee (pod) is always working the nginx shift.

**Technical Details:**
```
API server creates Deployment object
    ↓
Controller Manager sees it
    ↓
Creates ReplicaSet
    ↓
ReplicaSet Controller creates Pod
    ↓
Scheduler assigns to node
    ↓
kubelet starts container
```

---

### **Step 2.2: Verify Pod Running**

**Command:**
```bash
kubectl get pods -l app=nginx
```

**Expected Output:**
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-54c98b4f84-pp6km   1/1     Running   0          8s
```

**What This Proves:**
- ✅ Deployment controller working
- ✅ ReplicaSet controller working
- ✅ Scheduler assigned pod to node
- ✅ kubelet started container
- ✅ Container runtime (containerd) functioning

**Result:** ✅ PASSED - Deployment created and pod running

---

## **Test 3: Port Forwarding**

### **Step 3.1: Get Pod Name**

**Command:**
```bash
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

### **Step 3.2: Forward Local Port to Pod**

**Command:**
```bash
kubectl port-forward $POD_NAME 8080:80
```

**What It Does:**
Creates a tunnel from your local machine port 8080 to the pod's port 80.

**Analogy:**
Like running a private phone line directly to someone's desk, bypassing the main switchboard.

**Technical Details:**
```
kubectl
    ↓
API server
    ↓
Streaming connection to kubelet API
    ↓
kubelet proxies TCP to container's port 80
```

**Expected Output:**
```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

---

### **Step 3.3: Test the Tunnel**

**Command (from second terminal):**
```bash
curl --head http://127.0.0.1:8080
```

**Expected Output:**
```
HTTP/1.1 200 OK
Server: nginx/1.29.4
```

**Initial Result:** ❌ FAILED (see [Issues Encountered](#issues-encountered-and-resolution))

**Final Result:** ✅ PASSED (after certificate fix)

---

## **Test 4: Container Logs**

### **Step 4.1: Retrieve Logs**

**Command:**
```bash
kubectl logs $POD_NAME
```

**What It Does:**
Retrieves stdout/stderr logs from the container.

**Analogy:**
Reading a worker's activity log to see what they've been doing.

**Technical Details:**
```
kubectl
    ↓
API server
    ↓
kubelet API on worker node
    ↓
containerd
    ↓
Reads container's log file
```

**Expected Output:**
```
2026/01/18 17:03:44 [notice] 1#1: nginx/1.29.4
10.200.1.1 - - [18/Jan/2026:17:15:23 +0000] "HEAD / HTTP/1.1" 200 0
127.0.0.1 - - [18/Jan/2026:18:38:28 +0000] "HEAD / HTTP/1.1" 200 0
```

**Initial Result:** ❌ FAILED (same certificate error)

**Final Result:** ✅ PASSED (after certificate fix)

---

## **Test 5: Execute Commands in Container**

### **Step 5.1: Run Command**

**Command:**
```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

**What It Does:**
Runs a command inside the running container (like SSH into a container).

**Analogy:**
Calling an employee to ask them a question directly instead of checking their written reports.

**Technical Details:**
```
kubectl
    ↓
API server
    ↓
kubelet API
    ↓
containerd executes command via runc
    ↓
Streams output back
```

**Expected Output:**
```
nginx version: nginx/1.29.4
```

**Initial Result:** ❌ FAILED (same certificate error)

**Final Result:** ✅ PASSED (after certificate fix)

---

## **Test 6: Services (NodePort)**

### **Step 6.1: Expose Deployment as Service**

**Command:**
```bash
kubectl expose deployment nginx --port 80 --type NodePort
```

**What It Does:**
Creates a Service that exposes the nginx pod on a high port (30000-32767) on all nodes.

**Analogy:**
Opening a public phone number that routes to your employee's desk.

**Technical Details:**
```
Creates Service object
    ↓
kube-proxy watches
    ↓
Updates iptables rules on all nodes
    ↓
External traffic to NodePort routes to pod IP
```

---

### **Step 6.2: Test Service Access**

**Command:**
```bash
NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
NODE_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].spec.nodeName}")

curl -I http://${NODE_NAME}:${NODE_PORT}
```

**Expected Output:**
```
HTTP/1.1 200 OK
Server: nginx/1.29.4
```

**Why This Passed When Others Failed:**
NodePort traffic goes directly to the node's network interface, not through kubelet's TLS API.

**Result:** ✅ PASSED - Service networking works

---

## **Issues Encountered and Resolution**

### **The Problem: TLS Certificate Verification Failures**

Three tests failed (port-forward, logs, exec) with the same error:

```
tls: failed to verify certificate: x509: certificate is valid for 
node-1.kubernetes.local, not node-1
```

---

### **Root Cause Analysis**

**Investigation Step 1: Check Node Registration**
```bash
kubectl get nodes -o wide
```

Result: Nodes registered with **short hostnames**: `node-0`, `node-1`

**Investigation Step 2: Check Certificate SANs**
```bash
ssh root@node-1 "openssl x509 -in /var/lib/kubelet/kubelet.crt -text -noout | grep -A1 'Subject Alternative Name'"
```

Result:
```
X509v3 Subject Alternative Name:
    DNS:node-1.kubernetes.local, IP Address:10.240.0.21
```

**The Issue:**
```
┌─────────────────────────────────────────────────────────────────────┐
│                        Certificate Mismatch                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Node Registration         Certificate SANs                        │
│  ─────────────────         ────────────────                        │
│  node-0                    node-0.kubernetes.local ✓               │
│  node-1                    node-1.kubernetes.local ✓               │
│                            10.240.0.20/21 ✓                        │
│                            node-0 ✗ MISSING                        │
│                            node-1 ✗ MISSING                        │
│                                                                     │
│  When kubectl connects to kubelet API:                             │
│    - Uses node name from cluster registration (node-1)             │
│    - TLS verification fails (cert doesn't list "node-1")           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Why Service Test Passed:**
NodePort traffic goes directly to the node's network interface, not through kubelet's TLS API.

---

### **The Fix: Regenerate Certificates with Both Hostnames**

#### **Step 1: Backup and Update ca.conf**

```bash
# Backup original
cp ca.conf ca.conf.backup

# Add short hostnames to Subject Alternative Names
sed -i '/\[alt_names_kubelet-node-0\]/,/^\[/ {
  /DNS.1 = node-0.kubernetes.local/a\
DNS.2 = node-0
}' ca.conf

sed -i '/\[alt_names_kubelet-node-1\]/,/^\[/ {
  /DNS.1 = node-1.kubernetes.local/a\
DNS.2 = node-1
}' ca.conf
```

**Before:**
```ini
[alt_names_kubelet-node-0]
DNS.1 = node-0.kubernetes.local
IP.1 = 10.240.0.20
```

**After:**
```ini
[alt_names_kubelet-node-0]
DNS.1 = node-0.kubernetes.local
DNS.2 = node-0
IP.1 = 10.240.0.20
```

---

#### **Step 2: Regenerate Node Certificates**

**Node-0:**
```bash
openssl genrsa -out "node-0.key" 4096

openssl req -new -key "node-0.key" -sha256 \
  -config "ca.conf" -section node-0 \
  -out "node-0.csr"

openssl x509 -req -days 3653 -in "node-0.csr" \
  -copy_extensions copyall \
  -sha256 -CA "ca.crt" \
  -CAkey "ca.key" \
  -CAcreateserial \
  -out "node-0.crt"
```

**Node-1:**
```bash
openssl genrsa -out "node-1.key" 4096

openssl req -new -key "node-1.key" -sha256 \
  -config "ca.conf" -section node-1 \
  -out "node-1.csr"

openssl x509 -req -days 3653 -in "node-1.csr" \
  -copy_extensions copyall \
  -sha256 -CA "ca.crt" \
  -CAkey "ca.key" \
  -CAcreateserial \
  -out "node-1.crt"
```

---

#### **Step 3: Verify New Certificates**

```bash
openssl x509 -in node-0.crt -text -noout | grep -A2 'Subject Alternative Name'
openssl x509 -in node-1.crt -text -noout | grep -A2 'Subject Alternative Name'
```

**Output:**
```
X509v3 Subject Alternative Name:
    DNS:node-0.kubernetes.local, DNS:node-0, IP Address:10.240.0.20
X509v3 Subject Alternative Name:
    DNS:node-1.kubernetes.local, DNS:node-1, IP Address:10.240.0.21
```

✅ Both FQDN and short hostname now present

---

#### **Step 4: Deploy New Certificates to Worker Nodes**

```bash
# Backup old certificates
ssh root@node-0 "cp /var/lib/kubelet/kubelet.crt /var/lib/kubelet/kubelet.crt.old"
ssh root@node-0 "cp /var/lib/kubelet/kubelet.key /var/lib/kubelet/kubelet.key.old"
ssh root@node-1 "cp /var/lib/kubelet/kubelet.crt /var/lib/kubelet/kubelet.crt.old"
ssh root@node-1 "cp /var/lib/kubelet/kubelet.key /var/lib/kubelet/kubelet.key.old"

# Copy new certificates
scp node-0.crt root@node-0:/var/lib/kubelet/kubelet.crt
scp node-0.key root@node-0:/var/lib/kubelet/kubelet.key
scp node-1.crt root@node-1:/var/lib/kubelet/kubelet.crt
scp node-1.key root@node-1:/var/lib/kubelet/kubelet.key

# Restart kubelet to pick up new certificates
ssh root@node-0 "systemctl restart kubelet"
ssh root@node-1 "systemctl restart kubelet"
```

**Verify kubelets active:**
```bash
ssh root@node-0 "systemctl is-active kubelet"  # active
ssh root@node-1 "systemctl is-active kubelet"  # active
```

---

### **Second Issue: RBAC Permissions**

After fixing certificates, port-forward gave a new error:

```
error: error upgrading connection: unable to upgrade connection: 
Forbidden (user=kube-apiserver, verb=create, resource=nodes, subresource(s)=[proxy])
```

**Cause:**
The API server user (`kube-apiserver`) didn't have RBAC permission to proxy connections to nodes.

**Fix:**
```bash
kubectl create clusterrolebinding kube-apiserver-kubelet-admin \
  --clusterrole=system:kubelet-api-admin \
  --user=kube-apiserver
```

**What This Does:**
Grants the `kube-apiserver` user the `system:kubelet-api-admin` ClusterRole, which allows proxying to kubelet APIs.

---

## **Final Test Results**

All tests re-run after fixes:

| Test | Command | Result |
|------|---------|--------|
| 1. Data Encryption | `etcdctl get ... \| hexdump` | ✅ `k8s:enc:aescbc:v1:key1` |
| 2. Deployments | `kubectl get pods` | ✅ nginx pod Running |
| 3. Port Forwarding | `curl http://127.0.0.1:8080` | ✅ HTTP/1.1 200 OK |
| 4. Logs | `kubectl logs $POD_NAME` | ✅ nginx startup logs |
| 5. Exec | `kubectl exec ... -- nginx -v` | ✅ nginx version: nginx/1.29.4 |
| 6. Services | `curl http://${NODE_NAME}:${NODE_PORT}` | ✅ HTTP/1.1 200 OK |

---

## **What I Learned**

### **1. Certificate SANs Matter**

When generating TLS certificates for services accessed by multiple hostnames:
- Include **all possible hostnames** in Subject Alternative Names
- Include both FQDNs (`node-1.kubernetes.local`) and short names (`node-1`)
- Production clusters use certificate auto-rotation with comprehensive SANs

### **2. How kubectl Accesses Kubelets**

```
kubectl port-forward/logs/exec
    ↓
API Server (10.240.0.10:6443)
    ↓
Kubelet API on worker (node-1:10250) ← Uses hostname from node registration
    ↓
containerd/container
```

The kubelet connection uses TLS with certificate verification, requiring valid SANs.

### **3. RBAC for API Server**

The API server itself is a user (`kube-apiserver`) that needs permissions to access kubelet APIs for:
- Proxying (port-forward)
- Logs
- Exec commands

---

## **Production Differences**

| Aspect | KTHW (Tutorial) | Production |
|--------|-----------------|------------|
| **Certificates** | Manual generation/distribution | Automated (cert-manager, kubelet rotation) |
| **Certificate SANs** | Static, manual | Dynamic based on node discovery |
| **RBAC** | Manual configuration | Pre-configured for standard operations |
| **Monitoring** | Manual verification | Alerting for certificate expiration |
| **Testing** | Manual smoke tests | Automated in CI/CD pipelines |

### **Production Certificate Management:**
```
cert-manager
    ↓
Automatically issues certificates
    ↓
Monitors expiration
    ↓
Rotates before expiry
    ↓
No manual intervention
```

---

## **Troubleshooting**

### **Issue: "x509: certificate is valid for X, not Y"**

**Cause:** Certificate SANs don't include the hostname being used

**Check:**
```bash
openssl x509 -in <cert-file> -text -noout | grep -A2 'Subject Alternative Name'
```

**Fix:** Regenerate certificate with all required hostnames in SANs

---

### **Issue: "Forbidden (user=X, verb=Y, resource=Z)"**

**Cause:** RBAC permissions missing

**Check:**
```bash
kubectl auth can-i <verb> <resource> --as=<user>
```

**Fix:** Create appropriate ClusterRoleBinding

---

### **Issue: Pod stuck in Pending**

**Cause:** Scheduler can't place pod (resource constraints, taints, etc.)

**Check:**
```bash
kubectl describe pod <pod-name>
```

**Fix:** Check node resources, taints, and tolerations

---

### **Issue: "connection refused" on port-forward**

**Cause:** kubelet not running or not listening on port 10250

**Check:**
```bash
ssh root@<node> "systemctl status kubelet"
ssh root@<node> "ss -tlnp | grep 10250"
```

**Fix:** Restart kubelet, check kubelet logs

---

## **Section 12 Accomplishments**

### **Verified Components:**
- ✅ etcd encryption at rest (AES-CBC)
- ✅ Deployment controller
- ✅ ReplicaSet controller
- ✅ Scheduler
- ✅ kubelet
- ✅ containerd
- ✅ kube-proxy
- ✅ Pod networking
- ✅ Service networking

### **Issues Resolved:**
- ✅ Certificate SAN mismatch fixed
- ✅ RBAC permissions for API server added
- ✅ All 6 smoke tests passing

### **Skills Demonstrated:**
- ✅ TLS certificate debugging
- ✅ RBAC troubleshooting
- ✅ Understanding kubectl → API → kubelet flow

---

## **Key Takeaways**

1. **Smoke tests validate the full stack** - From encryption to networking to container execution

2. **Certificate SANs must match all access patterns** - Include both FQDNs and short hostnames

3. **RBAC applies to internal components too** - API server needs permissions to access kubelets

4. **Debugging TLS requires checking both ends** - Client expectations vs server certificate

5. **NodePort bypasses kubelet API** - Direct network access doesn't need kubelet TLS

6. **Production automates what I did manually** - cert-manager, pre-configured RBAC, monitoring

---

## **Summary**

Successfully validated the entire Kubernetes cluster with comprehensive smoke tests. Encountered and resolved a real-world TLS certificate issue, demonstrating the importance of proper certificate configuration and RBAC permissions in Kubernetes operations.

**All 6 smoke tests PASSED ✅**

---

## **Commands Reference**

```bash
# Test 1: Data Encryption
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"

ssh root@server \
  'etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C'

# Test 2: Deployments
kubectl create deployment nginx --image=nginx:latest
kubectl get pods -l app=nginx

# Test 3: Port Forwarding
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:80
curl --head http://127.0.0.1:8080  # In another terminal

# Test 4: Logs
kubectl logs $POD_NAME

# Test 5: Exec
kubectl exec -ti $POD_NAME -- nginx -v

# Test 6: Services
kubectl expose deployment nginx --port 80 --type NodePort
NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
NODE_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].spec.nodeName}")
curl -I http://${NODE_NAME}:${NODE_PORT}

# Certificate Fix Commands
openssl x509 -in <cert> -text -noout | grep -A2 'Subject Alternative Name'

# RBAC Fix
kubectl create clusterrolebinding kube-apiserver-kubelet-admin \
  --clusterrole=system:kubelet-api-admin \
  --user=kube-apiserver
```
