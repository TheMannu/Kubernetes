# 📘 Scenario 25: Orphaned PVs Causing Unscheduled Pods

**Category**: Storage Management  
**Environment**: Kubernetes 1.20, vSphere CSI Driver 2.3  
**Impact**: 47 PVCs stuck in Pending state, blocking application deployments  

---
## Scenario Summary  
PersistentVolumes (PVs) left in `Released` state after pod deletions prevented new PVCs from binding, causing critical workloads to fail scheduling.

---

## What Happened  
- **Storage lifecycle gap**:  
  - PVs created with `persistentVolumeReclaimPolicy: Retain`  
  - No cleanup process for released volumes  
- **Provisioning failure**:  
  - New PVCs for same storage class failed with `waiting for first consumer to be created`  
  - vSphere CSI logs showed `volume already exists` errors  
- **Cascading effects**:  
  - StatefulSet pods stuck in `Pending`  
  - Database initialization jobs timed out  

---
