# 📘 Scenario 36: Misconfigured NodeAffinity Excluding All Nodes

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.22, Azure AKS  
**Impact**: Critical application deployment failed for 6+ hours  

---

## Scenario Summary  
A deployment with overly restrictive `nodeAffinity` rules required non-existent node labels, making all cluster nodes invalid targets and preventing any pod scheduling.

---
zone=us-west-3`  
  - Cluster only had zones `us-west-1` and `us-west-2`  
- **Scheduling deadlock**:  
  - All pods stuck in `Pending` with `0/15 nodes available`  
  - `kubectl describe pod` showed `node(s) didn't match node selector`  
- **Business impact**:  
  - New feature deployment blocked  
  - Emergency rollback required but also failed (same affinity rules)  

---

## Diagnosis Steps  

### 1. Check pending pods:
```sh
kubectl get pods --all-namespaces --field-selector status.phase=Pending
# Showed 12 pods from affected deployment
```
