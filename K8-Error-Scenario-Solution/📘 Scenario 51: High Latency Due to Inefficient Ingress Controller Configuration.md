# 📘 Scenario 51: High Latency Due to Inefficient Ingress Controller Configuration

**Category**: Network Performance & Ingress Routing  
**Environment**: Kubernetes 1.20, AWS EKS with NGINX Ingress Controller  
**Impact**: 500ms+ latency increase, 30% error rate during peak traffic  

---

## Scenario Summary  
Inefficient Ingress routing configurations caused severe latency spikes and packet processing delays, degrading user experience and increasing error rates for external-facing applications.

---

## What Happened  
- **Routing complexity explosion**:  
  - 500+ ingress rules with complex regex path matching  
  - Default NGINX configuration with 100k+ `worker_connections` limit  
  - No caching or keep-alive optimizations enabled  
- **Performance degradation**:  
  - P95 latency increased from 50ms to 580ms  
  - NGINX worker processes at 100% CPU utilization  
  - `upstream request timed out` errors in application logs  
- **Configuration audit findings**:  
  - 80% of routes used regex matching instead of exact paths  
  - Missing HTTP/2 and gzip compression  
  - No rate limiting or connection pooling  

---