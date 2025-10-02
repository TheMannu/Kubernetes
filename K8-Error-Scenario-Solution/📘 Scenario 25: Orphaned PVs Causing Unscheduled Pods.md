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

## Diagnosis Steps  

### 1. Verify PVC status:
```sh
kubectl get pvc -A -o wide | grep -v Bound
# Showed multiple PVCs pending
```

### 2. Check PV inventory:
```sh
kubectl get pv --sort-by=.metadata.creationTimestamp \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.claimRef.name,RECLAIM:.spec.persistentVolumeReclaimPolicy
# Revealed 20+ PVs in Released state
```

### 3. Inspect storage provider:
```sh
kubectl logs -n kube-system ds/vsphere-csi-node -c vsphere-csi-driver | grep -A10 "already exists"
```

### 4. Audit reclaim policies:
```sh
kubectl get sc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.reclaimPolicy}{"\n"}{end}'
# Showed 80% volumes configured with Retain
```

---