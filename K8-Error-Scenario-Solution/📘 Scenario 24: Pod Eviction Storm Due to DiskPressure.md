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