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

### 3. Analyze pod termination:
```sh
kubectl get pods -n database -o wide | grep -E "Terminating|Pending"
kubectl describe pod cassandra-0 -n database | grep -A10 "Events"
# Showed waiting for volume detachment
```

### 4. Check pod disruption readiness:
```sh
kubectl get pods -n database -l app=cassandra -o json | \
  jq -r '.items[] | .metadata.name + " " + (.status.conditions[] | select(.type=="Ready") | .status)'
# Output: 1/3 pods not ready
```

---

## Root Cause  
**Overly conservative availability guarantees**:  
1. PDB `minAvailable` equal to replica count → zero disruption allowance  
2. Stateful pods with long termination grace periods  
3. No pod readiness for replacement scheduling  
4. Storage volume detachment delays  

---

## Fix/Workaround  

### Emergency Maintenance Acceleration:
```sh
# 1. Temporarily relax PDB (if SLA allows)
kubectl patch pdb cassandra-pdb -n database -p '{"spec":{"minAvailable":2}}'

