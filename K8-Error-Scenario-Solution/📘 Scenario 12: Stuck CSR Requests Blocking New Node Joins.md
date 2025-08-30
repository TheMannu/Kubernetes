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
