# 📘 Scenario 29: kube-scheduler Crash Due to Invalid Leader Election Config

**Category**: Control Plane Stability  
**Environment**: Kubernetes 1.24, Custom Scheduler Deployment  
**Impact**: Complete scheduling halt for 1.5 hours  

---

## Scenario Summary  
A misconfigured leader election namespace caused the `kube-scheduler` to crash on startup, freezing all new pod scheduling across the cluster.

---

## What Happened  
- **Helm customization error**:  
  - Values override set `leaderElection.namespace: kube-scheduler` (non-existent)  
  - Deployed via CI/CD pipeline without validation  
- **Immediate symptoms**:  
  - Scheduler pods crashed with `panic: failed to create leader election record`  
  - `kubectl get events` showed `FailedScheduling` for 300+ pods  
- **Configuration audit**:  
  - Leader election locks require namespace existence  
  - Default RBAC lacked namespace creation permissions  

---

## Diagnosis Steps  

### 1. Check scheduler status:
```sh
kubectl -n kube-system get pods -l component=kube-scheduler
# Showed CrashLoopBackOff
```

### 2. Inspect crash logs:
```sh
kubectl -n kube-system logs -l component=kube-scheduler --tail=100 | grep -A5 panic
# Output: "leases.coordination.k8s.io is forbidden: cannot create resource in namespace kube-scheduler"
```

### 3. Verify Helm values:
```sh
helm get values kube-scheduler -n kube-system | yq '.leaderElection'
# Showed namespace: kube-scheduler
```

### 4. Check namespace existence:
```sh
kubectl get ns kube-scheduler
# Error: NotFound
```

---

## Root Cause  
**Configuration mismatch**:  
1. Non-existent namespace specified for leader election lock  
2. No fallback to default `kube-system` behavior  
3. Missing pre-flight namespace validation  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# 1. Create missing namespace
kubectl create ns kube-scheduler

# 2. Patch Helm release
helm upgrade kube-scheduler -n kube-system --reuse-values \
  --set leaderElection.namespace=kube-system

# 3. Force pod restart
kubectl -n kube-system rollout restart deploy kube-scheduler
```

### Long-term Solution:
```yaml
# values.yaml hardening
leaderElection:
  namespace: kube-system  # Immutable
  resourceName: kube-scheduler
  resourceLock: leases
schedulerName: default-scheduler
```

---

---

## Lessons Learned  
⚠️ **Leader election is fragile**: Requires exact namespace/RBAC  
⚠️ **Helm defaults matter**: Should match upstream conventions  
⚠️ **Scheduler is critical**: Single component failure halts operations  

---
