# 📘 Scenario 12: Stuck CSR Requests Blocking New Node Joins

**Category**: Cluster Authentication  
**Environment**: Kubernetes v1.20, kubeadm cluster  
**Impact**: Node provisioning freeze, scaling operations failed  

---

## Scenario Summary  
A disabled CSR approval controller caused a backlog of 500+ unapproved certificate signing requests, preventing new nodes from joining the cluster and existing nodes from renewing certificates.  

---

## What Happened  
- **Security patch misstep**:  
  - `--cluster-signing-cert-file` flag was removed during CIS hardening  
  - CSR approval controller silently stopped processing requests  
- **Symptoms emerged**:  
  - New nodes stuck at `[kubelet] Waiting for server to sign certificate`  
  - Existing nodes began failing health checks as certificates expired  
  - `kubectl get nodes` showed `NotReady` for recently added nodes  
- **Certificate chain reaction**:  
  - Kubelet client certs (default 1y expiry) started expiring en masse  
  - Cluster-autoscaler couldn't provision worker nodes  

---

## Diagnosis Steps  

### 1. Check CSR backlog:
```sh
kubectl get csr -o wide | grep -c Pending
# Output showed 527 pending requests
```

### 2. Inspect kubelet logs:
```sh
journalctl -u kubelet --no-pager | grep -A10 "certificate"
# "Failed to request signed certificate: timed out waiting for CSR to be signed"
```

### 3. Verify controller status:
```sh
kubectl -n kube-system get pod -l component=kube-controller-manager \
  -o jsonpath='{.items[0].spec.containers[0].command}' | jq
# Missing --cluster-signing-cert-file flag
```

### 4. Check certificate expiry:
```sh
openssl x509 -enddate -noout -in /var/lib/kubelet/pki/kubelet-client-current.pem
# "notAfter=Apr 12 13:32:42 2023 GMT" (expired)
```

---

## Root Cause  
**Broken auto-approval flow**:  
1. Security hardening disabled certificate signing controller  
2. No monitoring for CSR approval latency  
3. Certificate expiry wave compounded the problem  

## Fix/Workaround  

### Emergency Recovery:
```sh
# Bulk approve node client CSRs
kubectl get csr -o name | grep 'node-client' | xargs kubectl certificate approve

# Approve all pending kubelet-serving CSRs
kubectl get csr -o json | \
  jq -r '.items[] | select(.status == {}) | .metadata.name' | \
  xargs kubectl certificate approve
```

### Controller Restoration:
```yaml
# /etc/kubernetes/manifests/kube-controller-manager.yaml
spec:
  containers:
  - command:
    - kube-controller-manager
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,csrapproving  # Explicitly enable
```

---

## Lessons Learned  
⚠️ **CSRs are cluster lifeblood**: Controls node auth and TLS rotations  
⚠️ **Silent failures**: Controllers fail open with no alerts  
⚠️ **Certificate expiry waves**: Synchronized issuance causes time-bomb effects  

---

## Prevention Framework  

### 1. Automated Approval Policies
```yaml
# CSR Approver ClusterRole (ensure it exists)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/nodeclient"]
  verbs: ["create"]
```

### 2. Monitoring
```yaml
# Critical Prometheus alerts
- alert: PendingCSRsHigh
  expr: count(kube_certificatesigningrequest_status_condition{condition="Approved",status="false"}) > 10
  for: 5m
  labels:
    severity: critical

- alert: CertificateExpirySoon
  expr: kubelet_certificate_manager_client_expiration_seconds < 86400*7  # 7 days
```

### 3. Automation Safeguards
```sh
# Pre-flight check for new nodes
kubeadm join --dry-run | grep -q "certificate signing request" || \
  { echo "CSR flow broken"; exit 1; }
```

### 4. Certificate Management
```yaml
# KubeletConfiguration (rotate more frequently)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
rotateCertificates: true
certificateRotationStrategy:
  expirationSeconds: 86400*30  # 30-day certs
```

---

**Key Metrics to Monitor**:  
- `kube_certificatesigningrequest_status_condition`  
- `kubelet_certificate_manager_client_expiration_seconds`  
- `controller_manager_ttl_after_finished_seconds`  

**Debugging Tools**:  
```sh
# Inspect CSR details
kubectl get csr -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.username}{"\t"}{.status.conditions[*].type}{"\n"}{end}'

# Verify CA certificate chain
openssl verify -CAfile /etc/kubernetes/pki/ca.crt /var/lib/kubelet/pki/kubelet-client-current.pem
```

**Renewal Procedure**:  
```sh
# Force certificate renewal (on node)
systemctl restart kubelet
rm /var/lib/kubelet/pki/kubelet-client-*
```

---