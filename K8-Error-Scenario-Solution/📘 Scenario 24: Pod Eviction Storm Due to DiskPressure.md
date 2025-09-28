# 📘 Scenario #24: Pod Eviction Storm Due to DiskPressure

**Category**: Node Resource Management  
**Environment**: Kubernetes 1.25, Self-managed, containerd runtime  
**Impact**: 83% of cluster pods evicted, 2-hour service disruption  

---

## Scenario Summary  
A mass container image update triggered uncontrolled disk consumption across all worker nodes, causing cascading pod evictions and cluster instability.

---