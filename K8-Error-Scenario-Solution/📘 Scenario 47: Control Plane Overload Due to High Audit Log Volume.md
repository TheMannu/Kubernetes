# 📘 Scenario 47: Control Plane Overload Due to High Audit Log Volume

**Category**: Control Plane Performance & Security  
**Environment**: Kubernetes 1.22, Azure AKS, 1000+ nodes  
**Impact**: API server latency increased to 15s+, cluster operations slowed to a crawl  

---

## Scenario Summary  
An overly permissive audit logging policy generated massive volumes of audit events, overwhelming the API server and etcd, causing severe performance degradation across the entire cluster.

---
