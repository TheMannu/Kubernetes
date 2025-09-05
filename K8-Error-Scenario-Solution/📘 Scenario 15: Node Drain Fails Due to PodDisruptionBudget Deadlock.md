# 📘 Scenario 15: Node Drain Fails Due to PodDisruptionBudget Deadlock

**Category**: Cluster Reliability  
**Environment**: Kubernetes v1.21, Production Cluster with HPA  
**Impact**: Unplanned maintenance delays, failed node rotations  

---

## Scenario Summary  
A `PodDisruptionBudget` (PDB) deadlock prevented node drainage when a deployment's `minAvailable` requirement exceeded available replicas during maintenance operations.

---

## What Happened  
- **Maintenance trigger**:  
  - Node required emergency patching (CVE-2021-44228)  
  - `kubectl drain` command hung indefinitely  
- **PDB conflict**:  
  - Deployment had `replicas: 2` with PDB `minAvailable: 2`  
  - Zero allowed disruptions (`kubectl get pdb` showed `ALLOWED-DISRUPTIONS: 0`) 
- **System behavior**:  
  - Drain operation timed out after 1h (`error when evicting pod: cannot evict as it would violate the pod's disruption budget`)  
  - Cluster autoscaler refused to scale up (CPU metrics below threshold)  

---