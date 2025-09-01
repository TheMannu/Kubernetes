# 📘 Scenario 13: Failed Cluster Upgrade Due to Unready Static Pods

**Category**: Control Plane Maintenance  
**Environment**: Kubernetes v1.21 → v1.23 upgrade, kubeadm  
**Impact**: Upgrade rollback required, 2-hour control plane outage  

---

## Scenario Summary  
A malformed etcd static pod manifest prevented the control plane from coming up during a minor version upgrade, requiring manual intervention and causing extended downtime.  

---

## What Happened  
- **Pre-upgrade change**:  
  - Admin modified `/etc/kubernetes/manifests/etcd.yaml` to add new volume  
  - Introduced typo in `volumeMounts` (`/var/lib/etcd` vs `/var/lib/etc`)  
- **Upgrade failure**:  
  - `kubeadm upgrade apply` hung at `[control-plane] Creating static Pod manifests`  
  - `kubelet` logged `Failed to create pod sandbox: invalid volume mount path`  
  - API server became unreachable after old etcd pod terminated  
- **Cascading effects**:  
  - Worker nodes marked control plane `NotReady` after 5m  
  - Scheduler stopped assigning new pods  

---
