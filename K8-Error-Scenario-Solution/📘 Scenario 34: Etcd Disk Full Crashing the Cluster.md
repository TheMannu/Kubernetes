# 📘 Scenario #34: Etcd Disk Full Crashing the Cluster

**Category**: Control Plane Stability  
**Environment**: Kubernetes 1.21, Self-managed with Local etcd  
**Impact**: Complete cluster outage lasting 3+ hours  

---

## Scenario Summary  
The etcd database exhausted all available disk space due to unmanaged growth of custom resources and lack of maintenance, causing complete control plane failure.

---

## What Happened  
- **Resource explosion**:  
  - Custom controller creating 10,000+ ConfigMaps daily  
  - No resource quotas or cleanup mechanisms  
- **Storage exhaustion**:  
  - `/var/lib/etcd` reached 100% capacity  
  - etcd logs showed `"mvcc: database space exceeded"`  
  - API server failed with `"etcdserver: no space"` errors  
- **Cascading failures**:  
  - Control plane components lost leader election  
  - Worker nodes marked `NotReady` after failed heartbeats  

---

## Diagnosis Steps  

### 1. Verify etcd status:
```sh
kubectl get pods -n kube-system -l component=etcd
# Showed CrashLoopBackOff
```

### 2. Check disk usage:
```sh
df -h /var/lib/etcd
# Output: 100% used
```
