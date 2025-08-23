# 📘 Scenario 6: Kubelet DiskPressure Loop on Large Image Pulls

**Category**: Node Resource Management  
**Environment**: Kubernetes v1.22, EKS  

## Scenario Summary  
Nodes entered a destructive eviction loop due to disk exhaustion from pulling oversized container images, triggering continuous `DiskPressure` conditions.

---

## What Happened  
- A **multi-layered container image (5GB+)** was deployed cluster-wide  
- Nodes exhibited:  
  - Frequent pod evictions (`Evicted: DiskPressure`)  
  - Cascading failures as evicted pods automatically rescheduled  
  - **CPU throttling** from kubelet's constant garbage collection  
- Cluster autoscaler added nodes but they quickly became unstable  

---

## Diagnosis Steps  

### 1. Verified node conditions:
```sh
kubectl get nodes -o json | jq '.items[].status.conditions'
# Output showed DiskPressure=True
```

### 2. Analyzed disk usage:
```sh
kubectl debug node/<node> -it --image=busybox -- df -h /var/lib/containerd
```

### 3. Inspected image cache:
```sh
kubectl debug node/<node> -it --image=ubuntu -- crictl images
# Showed multiple large images with duplicate layers
```

### 4. Checked kubelet logs:
```sh
journalctl -u kubelet --no-pager | grep -i "DiskPressure"
# Revealed continuous image garbage collection attempts
```

---

## Root Cause  
**Image storage bloat from**:  
1. **Unoptimized image** with redundant layers (dev tools left in production image)  
2. **No image size limits** in CI/CD pipeline  
3. **Insufficient disk headroom** for normal operations  

---
