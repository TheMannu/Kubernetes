# 📘 Scenario 52: Node Draining Delay During Maintenance

**Category**: Cluster Maintenance & Reliability  
**Environment**: Kubernetes 1.21, GKE with Stateful Workloads  
**Impact**: Extended maintenance window, delayed node recycling, increased cloud costs  

---

## Scenario Summary  
Strict PodDisruptionBudget (PDB) configurations prevented timely node draining during maintenance, causing prolonged downtime and operational delays.

---

## What Happened  
- **Planned node maintenance**:  
  - GKE node auto-upgrade triggered for security patches  
  - 5-node cluster with 3-node Cassandra ring  
- **Draining bottlenecks**:  
  - `kubectl drain` hung for 45+ minutes  
  - PDB `minAvailable: 3` for Cassandra with only 3 replicas  
  - Storage-attached pods with `terminationGracePeriodSeconds: 300`  
  - No healthy pods ready to replace terminating ones  
- **Impact**:  
  - Maintenance window extended by 2 hours  
  - Cascading delays for subsequent node rotations  
  - SLA violations for stateful services  

---

## Diagnosis Steps  

### 1. Check node draining status:
```sh
kubectl get nodes
kubectl describe node <node-being-drained> | grep -A10 "Conditions"
# Showed SchedulingDisabled but pods still running
```

### 2. Investigate PDB violations:
```sh
kubectl get pdb -A
kubectl describe pdb cassandra-pdb -n database
# Output: Allowed disruptions: 0
```
