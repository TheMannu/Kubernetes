# 📘 Scenario 36: Misconfigured NodeAffinity Excluding All Nodes

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.22, Azure AKS  
**Impact**: Critical application deployment failed for 6+ hours  

---

## Scenario Summary  
A deployment with overly restrictive `nodeAffinity` rules required non-existent node labels, making all cluster nodes invalid targets and preventing any pod scheduling.

---