# 📘 Scenario 14: Uncontrolled Logs Filled Disk on All Nodes

**Category**: Node Resource Management  
**Environment**: Kubernetes v1.24, AWS EKS, containerd runtime  
**Impact**: Cluster-wide disk pressure, pod evictions, and application failures  

---

## Scenario Summary  
A misconfigured debug flag in a production pod caused log explosions (100+ MB/sec), filling up `/var/log` across all worker nodes and triggering `DiskPressure` evictions.

---

## What Happened  
- **Accidental debug deployment**:  
  - `LOG_LEVEL=TRACE` committed to production configmap  
  - Deployed to 50+ replicas  
- **Storage symptoms**:  
  - `/var/log` reached 100% capacity in under 2 hours  
  - `kubelet` logged `No space left on device` errors  
  - Containerd became unresponsive (`failed to create task: write /var/log/...`)  
- **Cascading effects**:  
  - Node `NotReady` status due to failed health checks  
  - Cluster autoscaler spun up new nodes (which also filled rapidly)  

---

## Diagnosis Steps  

### 1. Identify disk usage:
```sh
kubectl debug node/<node> -it --image=alpine -- df -h /var/log
# Output: 100% used
```

### 2. Locate log offenders:
```sh
kubectl debug node/<node> -it --image=alpine -- \
  du -ah /var/log/containers/ | sort -rh | head -20
```

### 3. Verify pod logs:
```sh
kubectl logs <offending-pod> --tail=100 | grep -c "TRACE"
# Output: 100/100 lines at TRACE level
```

### 4. Check log rotation config:
```sh
kubectl debug node/<node> -it --image=alpine -- \
  cat /etc/docker/daemon.json | jq '.["log-opts"]'
# Showed missing size limits
```

---

## Root Cause  
**Unbounded logging**:  
1. No log rotation or size limits in container runtime config  
2. CI/CD allowed `TRACE` level in production  
3. Missing pod-level log quotas  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Free up space (on affected nodes)
kubectl debug node/<node> -it --image=alpine -- \
  truncate -s 0 /var/log/containers/*_<namespace>_<pod-name>*.log

# 2. Restart containerd
kubectl debug node/<node> -it --image=alpine -- \
  systemctl restart containerd

# 3. Scale down offender
kubectl scale deploy <debug-deployment> --replicas=0
```

### Long-term Solution:
```json
// /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd]
  disable_hugetlb_controller = false
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
  [plugins."io.containerd.grpc.v1.cri".containerd.logs]
    max_size = "100MB"
    max_files = 3
```
---
## Lessons Learned  
⚠️ **Logs are unmanaged resources**: Can consume 100% disk if unchecked  
⚠️ **Debug levels are dangerous**: `TRACE`/`DEBUG` must never reach production  
⚠️ **Node storage is shared**: One pod can take down entire node  

---

## Prevention Framework  

### 1. Runtime Log Policies
```yaml
# EKS AMI bootstrap configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
containerLogMaxSize: "50Mi"
containerLogMaxFiles: 3
``

### 2. Deployment Safeguards
```yaml
# OPA/Gatekeeper policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedLogLevels
metadata:
  name: deny-trace-logs
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    forbiddenLevels: ["TRACE", "DEBUG"]
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: NodeLogDiskFull
  expr: node_filesystem_avail_bytes{mountpoint="/var/log"} / node_filesystem_size_bytes{mountpoint="/var/log"} < 0.2
  for: 5m
  labels:
    severity: critical
```

### 4. CI/CD Validation
```sh
# Pre-deployment check
if grep -q "LOG_LEVEL=TRACE" $MANIFEST; then
  echo "ERROR: TRACE logging prohibited in production"
  exit 1
fi
```

---

**Key Metrics to Monitor**:  
- `container_fs_usage_bytes`  
- `node_filesystem_avail_bytes{mountpoint="/var/log"}`  
- `kubelet_evictions` by `DiskPressure`  

**Debugging Tools**:  
```sh
# Live log rate monitoring
kubectl exec <pod> -- sh -c "tail -f /proc/1/fd/1 | pv -lbtr >/dev/null"

# Per-pod log size
kubectl get pods -o json | jq '.items[] | {name:.metadata.name, logs:.spec.containers[].resources.limits."logging-size"}'
```
