# 📘 Scenario 26: Taints and Tolerations Mismatch Prevented Workload Scheduling

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.22, AKS GPU Node Pool  
**Impact**: GPU-accelerated workloads failed to schedule for 6+ hours  

---

## Scenario Summary  
New GPU nodes remained underutilized because critical workloads lacked required tolerations, causing scheduling deadlocks and resource starvation for AI/ML pipelines.

---
