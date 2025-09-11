# 📘 Scenario #17: Inconsistent Cluster State After Partial Backup Restore

**Category**: Disaster Recovery  
**Environment**: Kubernetes v1.24, Velero with etcd snapshots  
**Impact**: Production outage lasting 4+ hours due to incomplete state restoration  

---

## Scenario Summary  
A partial etcd restore operation created a "zombie cluster" state where API objects existed without their dependent resources (PVCs, Secrets), causing widespread pod failures.

---