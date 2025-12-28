# 📘 Scenario 46: Insufficient Resource Limits in Autoscaling Setup

**Category**: Autoscaling & Resource Management  
**Environment**: Kubernetes 1.21, GKE with HPA and Custom Metrics  
**Impact**: Application performance degradation under load due to failed autoscaling  

---

## Scenario Summary  
The Horizontal Pod Autoscaler failed to scale application pods during traffic spikes because resource limits were set too low, preventing accurate utilization calculations and triggering scaling actions.

---