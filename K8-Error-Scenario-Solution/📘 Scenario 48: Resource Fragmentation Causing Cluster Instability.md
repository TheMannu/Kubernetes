# 📘 Scenario 48: Resource Fragmentation Causing Cluster Instability

**Category**: Cluster Scheduling & Resource Optimization  
**Environment**: Kubernetes 1.23, Bare Metal with Heterogeneous Nodes  
**Impact**: Performance degradation, increased OOM kills, and failed deployments due to uneven resource distribution  

---

## Scenario Summary  
Resource fragmentation from imbalanced pod scheduling led to "hot" nodes experiencing performance issues while "cold" nodes remained underutilized, causing cluster-wide inefficiency and reliability problems.

---