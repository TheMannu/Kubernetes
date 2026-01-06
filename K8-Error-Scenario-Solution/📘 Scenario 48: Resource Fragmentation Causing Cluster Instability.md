# 📘 Scenario 48: Resource Fragmentation Causing Cluster Instability

**Category**: Cluster Scheduling & Resource Optimization  
**Environment**: Kubernetes 1.23, Bare Metal with Heterogeneous Nodes  
**Impact**: Performance degradation, increased OOM kills, and failed deployments due to uneven resource distribution  

---

## Scenario Summary  
Resource fragmentation from imbalanced pod scheduling led to "hot" nodes experiencing performance issues while "cold" nodes remained underutilized, causing cluster-wide inefficiency and reliability problems.

---

## What Happened  
- **Imbalanced scheduling over time**:  
  - Scheduler favored nodes with existing resource allocations (bin-packing algorithm)  
  - 80% of pods concentrated on 30% of nodes  
  - Node 1: 45 pods, 95% CPU utilization  
  - Node 2: 8 pods, 15% CPU utilization  
- **Observed symptoms**:  
  - High-latency API responses from overloaded nodes  
  - Frequent OOM kills on "hot" nodes  
  - Failed pod scheduling: `0/1 nodes available: Insufficient cpu` despite overall cluster capacity  
  - Uneven wear on hardware components  
- **Root analysis**:  
  - No pod anti-affinity rules for same-application pods  
  - Missing topology spread constraints  
  - Default scheduler scoring favored packing over distribution  

---
