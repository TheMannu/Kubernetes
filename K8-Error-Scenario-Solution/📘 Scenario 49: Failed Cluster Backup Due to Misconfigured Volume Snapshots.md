# 📘 Scenario 49: Failed Cluster Backup Due to Misconfigured Volume Snapshots

**Category**: Disaster Recovery & Storage  
**Environment**: Kubernetes 1.21, AWS EKS with EBS CSI Driver  
**Impact**: Critical production backup failure, risking data loss for RPO of 24 hours  

---

## Scenario Summary  
A misconfigured EBS CSI snapshot driver caused complete backup failures, leaving the cluster without viable restore points for critical stateful workloads.

---
