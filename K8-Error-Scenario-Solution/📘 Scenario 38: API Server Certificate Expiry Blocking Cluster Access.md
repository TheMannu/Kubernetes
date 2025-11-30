# 📘 Scenario 38: API Server Certificate Expiry Blocking Cluster Access

**Category**: Control Plane Security  
**Environment**: Kubernetes 1.19, kubeadm  
**Impact**: Complete cluster inaccessibility for 2+ hours  

---

## Scenario Summary  
The API server's TLS certificate expired after 1 year, rendering the entire cluster inaccessible to all clients (`kubectl`, kubelets, controllers) and breaking all cluster operations.

---

## What Happened  
- **Certificate lifecycle failure**:  
  - kubeadm-generated certificates reached 1-year expiration  
  - No automatic renewal process in place  
  - Control plane operating for 395 days without certificate rotation  
- **Complete access loss**:  
  - `kubectl` commands failed with `x509: certificate has expired`  
  - Worker nodes lost connection to API server (`NodeNotReady`)  
  - Controller managers unable to sync resources  
- **Silent degradation**:  
  - No warnings until exact expiration moment  
  - All components failed simultaneously at expiration time  

---

## Diagnosis Steps  

### 1. Verify cluster accessibility:
```sh
kubectl get nodes
# Error: "Unable to connect to the server: x509: certificate has expired or is not yet valid"
```

### 2. Check certificate expiration:
```sh
# On control plane node
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
# Output: "notAfter=May 15 14:22:00 2023 GMT" (expired)
```

### 3. Inspect API server logs:
```sh
journalctl -u kube-apiserver --no-pager -n 50 | grep -i cert
# Showed TLS handshake failures
```

### 4. Verify all critical certificates:
```sh
kubeadm certs check-expiration
# Revealed multiple expired certificates
```

---
---
