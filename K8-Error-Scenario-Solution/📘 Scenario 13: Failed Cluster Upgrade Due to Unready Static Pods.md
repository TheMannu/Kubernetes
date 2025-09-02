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

### Complete Upgrade:
```sh
kubeadm upgrade apply v1.23.5 --force
```

## Lessons Learned  
⚠️ **Static pods are fragile**: No admission controllers validate them  
⚠️ **kubelet is silent**: Errors only appear in logs, not `kubectl`  
⚠️ **Control plane domino effect**: etcd failure → API failure → Cluster failure  

---

## Prevention Framework  

### 1. Manifest Management
```sh
# Pre-change validation script
validate_manifest() {
  yamllint $1 && \
  kubeval --strict $1 --schema-location https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master
}
```

### 2. Change Control
```yaml
# Ansible pre-upgrade checklist
- name: Backup static manifests
  copy:
    src: "/etc/kubernetes/manifests/{{ item }}"
    dest: "/etc/kubernetes/manifests/{{ item }}.bak-{{ ansible_date_time.iso8601 }}"
  loop:
    - etcd.yaml
    - kube-apiserver.yaml
```

### 3. Monitoring
```yaml
# Prometheus alerts for static pods
- alert: StaticPodNotReady
  expr: time() - kube_pod_start_time{created_by_kind="Node"} > 300
  for: 2m
  labels:
    severity: critical
```

### 4. Upgrade Safeguards
```sh
# kubeadm pre-flight check
kubeadm upgrade diff --config=kubeadm-config.yaml
```

---

**Key Metrics to Monitor**:  
- `kube_pod_status_ready{created_by_kind="Node"}`  
- `kubelet_running_pods`  
- `etcd_server_has_leader`  

**Debugging Tools**:  
```sh
# Live static pod inspection
crictl inspectp $(crictl pods --name etcd -q) | jq '.status.conditions'

# Compare manifests
diff -u /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/manifests/etcd.yaml.bak
```

**Backup Procedure**:  
```sh
# Daily manifest backup
tar czf /backups/k8s-manifests-$(date +%F).tgz /etc/kubernetes/manifests
```

