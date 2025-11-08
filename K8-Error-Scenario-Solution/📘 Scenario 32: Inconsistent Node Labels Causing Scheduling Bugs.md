# 📘 Scenario 32: Inconsistent Node Labels Causing Scheduling Bugs

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.24, Multi-zone GKE  
**Impact**: Critical workloads failed to schedule, breaking zone-balancing guarantees  

---

## Scenario Summary  
Missing `topology.kubernetes.io/zone` labels on manually provisioned nodes caused topology-aware workloads to fail scheduling, violating high-availability requirements.

---