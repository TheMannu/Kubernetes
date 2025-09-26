# 📘 Scenario 23: CoreDNS CrashLoop Due to Invalid ConfigMap Update

**Category**: Cluster Networking  
**Environment**: Kubernetes 1.23, GKE (Managed Control Plane)  
**Impact**: Cluster-wide DNS outage lasting 47 minutes  

---

## Scenario Summary  
A malformed CoreDNS ConfigMap update caused all DNS resolution to fail, breaking service discovery and pod-to-pod communication across the entire cluster.

---

## What Happened  
- **Config change**:  
  - Added rewrite rule with incorrect syntax: `rewrite stop name regex (.*)\.internal internal.svc.cluster.local`  
  - Missing plugin declaration in Corefile preamble  
- **Immediate impact**:  
  - CoreDNS pods entered `CrashLoopBackOff`  
  - `kubectl logs` showed `Corefile:5 - Error during parsing: Unknown directive 'rewrit'`  
- **Cascading failures**:  
  - Service mesh (Istio) sidecars failed health checks  
  - `kubelet` reported `node not ready` due to DNS timeouts  

---
