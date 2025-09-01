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

## Diagnosis Steps  

### 1. Verify static pod status:
```sh
# On control plane node:
crictl pods --name etcd
# Showed "NotReady" state
```

### 2. Inspect kubelet logs:
```sh
journalctl -u kubelet -n 50 --no-pager | grep -A10 "static pod"
# Error: "invalid volume mount path: /var/lib/etc"
```

### 3. Validate manifest syntax:
```sh
yamllint /etc/kubernetes/manifests/etcd.yaml
# Passed (syntax valid but path incorrect)
```

### 4. Check etcd container:
```sh
crictl inspect $(crictl ps -a --name etcd -q) | jq '.status.reason'
# "CreateContainerError"
```

---

## Root Cause  
**Manifest validation gap**:  
1. **No pre-flight validation** of static pod changes  
2. **Case-sensitive path** typo went undetected  
3. **kubelet retry logic** didn't surface error clearly  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Restore from backup manifest
cp /etc/kubernetes/manifests/etcd.yaml.bak /etc/kubernetes/manifests/etcd.yaml

# 2. Force kubelet reload
systemctl restart kubelet

# 3. Verify etcd recovery
crictl logs $(crictl ps --name etcd -q)
```
