# 📘 Scenario #20: Node Labels Accidentally Overwritten by DaemonSet

**Category**: Cluster Configuration  
**Environment**: Kubernetes v1.24, Label Management DaemonSet  
**Impact**: Critical GPU workloads unscheduled for 3+ hours  

---

## Scenario Summary  
A well-intentioned but overly aggressive DaemonSet overwrote critical node labels (`gpu=true`, `storage=ssd`), disrupting scheduler decisions and causing workload placement failures.

---