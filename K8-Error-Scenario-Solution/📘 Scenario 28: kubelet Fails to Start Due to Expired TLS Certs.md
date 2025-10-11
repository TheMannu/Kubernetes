# 📘 Scenario 28: kubelet Fails to Start Due to Expired TLS Certs

**Category**: Cluster Authentication  
**Environment**: Kubernetes 1.19, kubeadm, On-prem  
**Impact**: 40% nodes remained `NotReady` after maintenance window  

---

## Scenario Summary  
Nodes failed to rejoin the cluster after a scheduled reboot when kubelet client certificates expired during the downtime, breaking authentication with the API server.

---

## What Happened  
- **Planned maintenance**:  
  - 2-hour power maintenance for rack PDU upgrade  
  - 18 nodes powered off during certificate rotation window  
- **Post-reboot failures**:  
  - `kubelet` crashed with `x509: certificate has expired or is not yet valid`  
  - `journalctl` showed `Failed to start kubelet: unable to load client certificate`  
- **Certificate analysis**:  
  - `kubelet-client-current.pem` expired 37 minutes into downtime  
  - No automatic rotation attempted due to power-off state  

---

## Diagnosis Steps  

### 1. Verify node status:
```sh
kubectl get nodes -o json | \
  jq -r '.items[] | select(.status.conditions[].reason=="KubeletNotReady") | .metadata.name'
```

### 2. Check kubelet logs:
```sh
journalctl -u kubelet --no-pager -n 50 | grep -i cert
# Output: "certificate expired on 2023-05-15 14:22:00 +0000 UTC"
```

### 3. Inspect certificates:
```sh
openssl x509 -enddate -noout -in /var/lib/kubelet/pki/kubelet-client-current.pem
# Output: "notAfter=May 15 14:22:00 2023 GMT"
```

### 4. Verify rotation config:
```sh
kubectl -n kube-system get cm kubelet-config -o json | \
  jq -r '.data."kubelet.conf"' | grep rotate
# Showed rotation enabled but node offline during window
```

---

## Root Cause  
**Certificate lifecycle gap**:  
1. 1-year certs issued at cluster init  
2. No renewal during 3-month maintenance blackout  
3. kubelet couldn't bootstrap without valid credentials  

---
