# 📘 Scenario 38: API Server Certificate Expiry Blocking Cluster Access

**Category**: Control Plane Security  
**Environment**: Kubernetes 1.19, kubeadm  
**Impact**: Complete cluster inaccessibility for 2+ hours  

---

## Scenario Summary  
The API server's TLS certificate expired after 1 year, rendering the entire cluster inaccessible to all clients (`kubectl`, kubelets, controllers) and breaking all cluster operations.

---
