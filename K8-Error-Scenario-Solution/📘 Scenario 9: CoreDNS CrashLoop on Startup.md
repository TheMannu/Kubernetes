# 📘 Scenario 9: CoreDNS CrashLoop on Startup  

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.24, DigitalOcean  
**Impact**: Cluster-wide DNS resolution failure  

---

## Scenario Summary  
CoreDNS pods entered a `CrashLoopBackOff` state due to an invalid `Corefile` configuration, breaking DNS resolution across all cluster workloads.  

---

## What Happened  
- **Custom `Corefile` update**:  
  - A new `rewrite` rule was added with incorrect syntax  
  - Applied via `kubectl edit configmap/coredns -n kube-system`  
- **Immediate failure**:  
  - CoreDNS pods crashed on startup (`Error in Corefile: invalid directive`)  
  - Cluster services began failing with `NXDOMAIN` errors  
- **Cascading effects**:  
  - `kubectl` commands slowed due to DNS timeouts  
  - Service mesh (Istio) sidecars failed health checks  

---