# 📘 Scenario #24: Pod Eviction Storm Due to DiskPressure

**Category**: Node Resource Management  
**Environment**: Kubernetes 1.25, Self-managed, containerd runtime  
**Impact**: 83% of cluster pods evicted, 2-hour service disruption  

---

## Scenario Summary  
A mass container image update triggered uncontrolled disk consumption across all worker nodes, causing cascading pod evictions and cluster instability.

---

## What Happened  
- **Batch update initiated**:  
  - Deployment rollout of 5GB image to 1,200 pods  
  - Simultaneous pulls across 50 nodes  
- **Storage exhaustion**:  
  - `/var/lib/containerd` reached 100% utilization in 8 minutes  
  - `kubelet` enforced `DiskPressure` evictions (`nodefs.available<10%`)  
- **Cascading failures**:  
  - System pods (CNI, CSI) evicted first  
  - Cluster-autoscaler spun up new nodes (which also filled)  
  - etcd overwhelmed by pod churn events  

---

## Diagnosis Steps  

### 1. Verify node conditions:
```sh
kubectl get nodes -o json | jq -r '.items[] | .metadata.name + " " + (.status.conditions[] | select(.type=="DiskPressure") | .status'
```

### 2. Check disk usage:
```sh
kubectl debug node/<node> -it --image=alpine -- df -h /var/lib/containerd
# Output: 100% used
```

### 3. Inspect image cache:
```sh
kubectl debug node/<node> -it --image=ubuntu -- crictl images --digests
# Showed 40+ GB of images
```

### 4. Analyze eviction logs:
```sh
journalctl -u kubelet --no-pager -n 100 | grep -A10 "DiskPressure"
# Logged "Attempting to reclaim ephemeral-storage"
```

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Pause deployments
kubectl rollout pause deploy/<batch-job>

# 2. Manual image pruning (on affected nodes)
crictl rmi --prune

# 3. Taint nodes to prevent rescheduling
kubectl taint nodes <node> disk-pressure=cleanup:NoSchedule
```

### Long-term Solution:
```yaml
# Kubelet configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  nodefs.available: "15%"
  imagefs.available: "20%"
evictionMinimumReclaim:
  nodefs.available: "5Gi"
  imagefs.available: "3Gi"
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
```

---

## Lessons Learned  
⚠️ **DiskPressure is non-forgiving**: Evictions start before OOM kills  
⚠️ **Storage is a shared resource**: One pod can starve the node  
⚠️ **Default thresholds are dangerous**: Must be tuned per workload  

---

