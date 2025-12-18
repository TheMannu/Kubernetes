# 📘 Scenario 43: Node Pool Scaling Impacting StatefulSets

**Category**: Storage & Scheduling  
**Environment**: Kubernetes 1.24, GKE with Regional Clusters  
**Impact**: StatefulSet pod disruptions causing database corruption and application downtime  

---

## Scenario Summary  
Automatic node pool scaling triggered involuntary StatefulSet pod migrations, breaking persistent volume attachments and causing data consistency issues in stateful workloads.

---
