# 📘 Scenario 10: Control Plane Unavailable After Flannel Misconfiguration  

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.18, On-prem (Bare Metal), Flannel CNI  
**Impact**: Complete control plane isolation, workload communication breakdown  

---

## Scenario Summary  
A new node joined with an incorrect pod CIDR, causing Flannel routing tables to corrupt and severing all control plane communications.  

---

## What Happened  
- **Node onboarding error**:  
  - New node configured with `10.244.1.0/24` when cluster expected `10.244.0.0/16`  
  - Flannel's `kube-subnet-mgr` failed to reconcile the mismatch  
- **Network symptoms**:  
  - API server became unreachable from worker nodes (`kubectl` timeout)  
  - Cross-node pod communication failed (`No route to host`)  
  - Flannel pods logged `"Failed to acquire lease"` errors  
- **Cascading failures**:  
  - Kube-proxy iptables rules became inconsistent  
  - Node status updates stopped flowing to control plane  

---
