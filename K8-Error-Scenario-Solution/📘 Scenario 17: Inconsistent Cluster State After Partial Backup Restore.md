# 📘 Scenario #17: Inconsistent Cluster State After Partial Backup Restore

**Category**: Disaster Recovery  
**Environment**: Kubernetes v1.24, Velero with etcd snapshots  
**Impact**: Production outage lasting 4+ hours due to incomplete state restoration  

---

## Scenario Summary  
A partial etcd restore operation created a "zombie cluster" state where API objects existed without their dependent resources (PVCs, Secrets), causing widespread pod failures.

---

## What Happened  
- **Disaster recovery scenario**:  
  - etcd corruption required restore from 6-hour-old Velero snapshot  
  - Only etcd data was restored (excluded PV snapshots and external Secrets)  
- **Post-restore symptoms**:  
  - 60% of pods stuck in `CreateContainerError`  
  - Persistent volume claims showed `<none>` in `kubectl get pvc -A`  
  - CSI driver logs reported `volume not found` errors  
- **Dependency graph breaks**:  
  - Deployments referenced non-existent PVCs  
  - ServiceAccounts lacked corresponding Secrets  

---
