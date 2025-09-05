# 📘 Scenario 15: Node Drain Fails Due to PodDisruptionBudget Deadlock

**Category**: Cluster Reliability  
**Environment**: Kubernetes v1.21, Production Cluster with HPA  
**Impact**: Unplanned maintenance delays, failed node rotations  

---

## Scenario Summary  
A `PodDisruptionBudget` (PDB) deadlock prevented node drainage when a deployment's `minAvailable` requirement exceeded available replicas during maintenance operations.

---
