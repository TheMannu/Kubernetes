# 📘 Scenario 13: Failed Cluster Upgrade Due to Unready Static Pods

**Category**: Control Plane Maintenance  
**Environment**: Kubernetes v1.21 → v1.23 upgrade, kubeadm  
**Impact**: Upgrade rollback required, 2-hour control plane outage  

---

## Scenario Summary  
A malformed etcd static pod manifest prevented the control plane from coming up during a minor version upgrade, requiring manual intervention and causing extended downtime.  

---
