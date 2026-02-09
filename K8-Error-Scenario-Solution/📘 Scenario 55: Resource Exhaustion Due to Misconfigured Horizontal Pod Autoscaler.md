# 📘 Scenario 55: Resource Exhaustion Due to Misconfigured Horizontal Pod Autoscaler

**Category**: Autoscaling & Resource Management  
**Environment**: Kubernetes 1.22, AWS EKS, 100-node cluster  
**Impact**: Cluster scaled to 150% capacity, 50% cost overrun, performance degradation  

---

## Scenario Summary  
An overly aggressive HPA configuration triggered runaway scaling, exhausting cluster resources and driving up cloud costs while actually degrading application performance due to resource contention.

---
