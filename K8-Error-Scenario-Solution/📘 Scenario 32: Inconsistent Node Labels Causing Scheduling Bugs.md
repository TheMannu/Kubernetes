# 📘 Scenario 32: Inconsistent Node Labels Causing Scheduling Bugs

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.24, Multi-zone GKE  
**Impact**: Critical workloads failed to schedule, breaking zone-balancing guarantees  

---

## Scenario Summary  
Missing `topology.kubernetes.io/zone` labels on manually provisioned nodes caused topology-aware workloads to fail scheduling, violating high-availability requirements.

---

## What Happened  
- **Manual node provisioning**:  
  - Emergency capacity added via `gcloud compute instances create`  
  - Missing cloud-controller-manager auto-labeling  
- **Scheduling failures**:  
  - Pods with `topologySpreadConstraints` stuck in `Pending`  
  - `kubectl describe pod` showed `0/3 nodes are available: 3 node(s) missing required label`  
- **HA impact**:  
  - 5-node cluster had only 2 labeled zones instead of 3  
  - StatefulSet pods concentrated in single zone  

---

## Diagnosis Steps  

### 1. Identify pending pods:
```sh
kubectl get pods -A --field-selector status.phase=Pending -o wide
```

### 2. Check scheduling events:
```sh
kubectl describe pod <pending-pod> | grep -A10 "Events"
# Output: "0/3 nodes are available: 3 node(s) missing required label topology.kubernetes.io/zone"
```


### 3. Audit node labels:
```sh
kubectl get nodes -o json | jq -r '.items[] | .metadata.name + " " + (.metadata.labels | to_entries | map(select(.key | contains("zone"))) | map("\(.key)=\(.value)") | join(","))'
# Showed 3 nodes without zone labels
```

### 4. Verify topology constraints:
```sh
kubectl get pods -o json | jq -r '.items[] | select(.spec.topologySpreadConstraints) | .metadata.name'
```

---
