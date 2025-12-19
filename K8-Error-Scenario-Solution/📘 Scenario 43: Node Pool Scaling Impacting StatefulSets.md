# 📘 Scenario 43: Node Pool Scaling Impacting StatefulSets

**Category**: Storage & Scheduling  
**Environment**: Kubernetes 1.24, GKE with Regional Clusters  
**Impact**: StatefulSet pod disruptions causing database corruption and application downtime  

---

## Scenario Summary  
Automatic node pool scaling triggered involuntary StatefulSet pod migrations, breaking persistent volume attachments and causing data consistency issues in stateful workloads.

---

## What Happened  
- **Cluster autoscaling event**:  
  - GKE autoscaler removed 3 nodes during cost optimization cycle  
  - StatefulSet pods using `volumeBindingMode: WaitForFirstConsumer`  
  - PVCs remained bound to original nodes' zones  
- **Failure symptoms**:  
  - StatefulSet pods stuck in `Pending` with `volume node affinity conflict`  
  - Database pods (Cassandra, PostgreSQL) lost quorum  
  - `kubectl describe pvc` showed `no available persistent volumes`  
- **Data integrity risks**:  
  - Multi-node databases experienced split-brain scenarios  
  - Write-ahead logs became inconsistent  

---

## Diagnosis Steps  

### 1. Identify affected StatefulSets:
```sh
kubectl get statefulsets -A -o json | \
  jq -r '.items[] | select(.spec.volumeClaimTemplates) | .metadata.namespace + "/" + .metadata.name'
```

### 2. Check PVC binding status:
```sh
kubectl get pvc -A -o wide | grep -v Bound
# Showed multiple PVCs in Pending state
```

### 3. Inspect volume affinity errors:
```sh
kubectl describe pvc data-cassandra-0 -n database | grep -A5 Events
# Output: "no persistent volumes available for this claim due to storage class node affinity conflict"
```

### 4. Verify node topology:
```sh
kubectl get nodes -o json | jq -r '.items[] | .metadata.labels."topology.kubernetes.io/zone"'
# Showed missing zones after scaling
```

---

## Root Cause  
**Topology-aware volume constraints**:  
1. Regional StorageClass with `volumeBindingMode: WaitForFirstConsumer`  
2. No pod anti-affinity or node affinity for StatefulSet pods  
3. PVCs bound to specific zones that became unavailable  

---
