# 📘 Scenario 20: Node Labels Accidentally Overwritten by DaemonSet

**Category**: Cluster Configuration  
**Environment**: Kubernetes v1.24, Label Management DaemonSet  
**Impact**: Critical GPU workloads unscheduled for 3+ hours  

---

## Scenario Summary  
A well-intentioned but overly aggressive DaemonSet overwrote critical node labels (`gpu=true`, `storage=ssd`), disrupting scheduler decisions and causing workload placement failures.

---

## What Happened  
- **Automated label deployment**:  
  - DaemonSet designed to standardize zone labels (`zone=us-east-1a`)  
  - Used `kubectl label --overwrite` instead of strategic merge  
- **Immediate symptoms**:  
  - GPU-requiring pods stuck in `Pending` (`0/3 nodes available: 3 node(s) didn't match Pod's node affinity`)  
  - Storage-sensitive workloads scheduled to HDD nodes  
- **Configuration drift**:  
  - 28 nodes lost 5+ custom labels each  
  - Scheduler metrics showed `FailedScheduling` events spiked 400%  

---
