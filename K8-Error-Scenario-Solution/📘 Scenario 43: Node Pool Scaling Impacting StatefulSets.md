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
