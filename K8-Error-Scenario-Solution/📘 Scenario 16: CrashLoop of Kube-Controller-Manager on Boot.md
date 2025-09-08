# 📘 Scenario #16: CrashLoop of Kube-Controller-Manager on Boot

**Category**: Control Plane Stability  
**Environment**: Kubernetes v1.23, Self-hosted Control Plane  
**Impact**: Cluster-wide controller failures (deployments, services stalled)  

---

## Scenario Summary  
The `kube-controller-manager` entered a crashloop after a cluster upgrade due to an obsolete admission plugin in its startup configuration, paralyzing core cluster operations.

---

## What Happened  
- **Post-upgrade failure**:  
  - Control plane pods restarted after `kubeadm upgrade` to v1.23  
  - `kube-controller-manager` crashed immediately with `unknown admission plugin` error  
- **System impact**:  
  - No new deployments could be created (`No controllers available to schedule`)  
  - Existing deployments stopped scaling (`Failed to list *v1.ReplicaSet`)  
- **Configuration mismatch**:  
  - Legacy `--enable-admission-plugins=NamespaceLifecycle,InitialResources` flag  
  - `InitialResources` plugin removed in Kubernetes 1.22  

---

## Diagnosis Steps  

### 1. Check controller-manager status:
```sh
kubectl -n kube-system get pod -l component=kube-controller-manager
# Showed CrashLoopBackOff
```

### 2. Inspect logs:
```sh
kubectl -n kube-system logs --tail=50 -l component=kube-controller-manager
# Error: "unknown admission plugin \"InitialResources\""
```

### 3. Verify version compatibility:
```sh
kube-controller-manager --version | grep -q "v1.23" || \
  echo "Version mismatch"
```

### 4. Check startup flags:
```sh
ps aux | grep kube-controller-manager | grep -o "\-\-enable\-admission\-plugins=[^ ]*"
# Output: --enable-admission-plugins=NamespaceLifecycle,InitialResources
```

## Root Cause  
**Breaking change unhandled**:  
1. Admission plugin `InitialResources` removed in v1.22 (unnoticed during upgrade planning)  
2. No pre-flight validation of controller-manager flags  
3. Static pod manifest not version-controlled  

---
