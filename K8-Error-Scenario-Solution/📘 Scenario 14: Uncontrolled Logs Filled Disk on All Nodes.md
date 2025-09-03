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
