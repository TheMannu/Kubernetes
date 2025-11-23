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

### 2. Inspect scheduling events:
```sh
kubectl describe pod <pending-pod> | grep -A20 Events
# Output: "0/15 nodes are available: 15 node(s) didn't match node affinity rules"
```

### 3. Analyze affinity configuration:
```sh
kubectl get deployment <name> -o yaml | yq '.spec.template.spec.affinity'
# Revealed invalid zone requirement
```

### 4. Verify available node labels:
```sh
kubectl get nodes -o json | jq -r '.items[].metadata.labels | 
  ."topology.kubernetes.io/zone"' | sort | uniq
# Output: us-west-1, us-west-2 (no us-west-3)
```

---

## Root Cause  
**Affinity validation gap**:  
1. Hard `requiredDuringScheduling` constraints with invalid values  
2. No pre-deployment validation of node labels  
3. Copy-paste error from different environment configuration  

---