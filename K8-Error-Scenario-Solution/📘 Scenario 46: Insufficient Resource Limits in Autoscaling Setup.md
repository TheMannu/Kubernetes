# 📘 Scenario 46: Insufficient Resource Limits in Autoscaling Setup

**Category**: Autoscaling & Resource Management  
**Environment**: Kubernetes 1.21, GKE with HPA and Custom Metrics  
**Impact**: Application performance degradation under load due to failed autoscaling  

---

## Scenario Summary  
The Horizontal Pod Autoscaler failed to scale application pods during traffic spikes because resource limits were set too low, preventing accurate utilization calculations and triggering scaling actions.

---

## What Happened  
- **Traffic surge event**:  
  - 5x normal traffic during peak hours  
  - Application latency increased from 50ms to 2s  
- **HPA failure analysis**:  
  - HPA status showed `CurrentReplicas: 2, DesiredReplicas: 2` despite 90% CPU utilization  
  - Resource metrics showed `100m` CPU limit per pod (0.1 CPU cores)  
  - Actual usage per pod: `95m` (95% of limit)  
  - HPA target utilization set to 80%, but calculation used `requests`, not actual usage  
- **Performance impact**:  
  - Queuing and dropped requests  
  - Database connection pool exhaustion  
  - User experience degradation  

---
