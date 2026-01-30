📘 Scenario 53: Unresponsive Cluster After Large-Scale Deployment

**Category**: Cluster Scaling & Deployment Strategies  
**Environment**: Kubernetes 1.19, Azure AKS, 50-node cluster  
**Impact**: Complete cluster unresponsiveness for 45 minutes, failed CI/CD pipelines  

---

## Scenario Summary  
A massive batch deployment of 500 pods simultaneously overwhelmed the control plane and etcd, causing cluster-wide API timeouts and rendering the cluster completely unresponsive.

---

## What Happened  
- **Aggressive deployment**:  
  - CI/CD pipeline deployed 500 identical pods in single `kubectl apply`  
  - Each pod requested 1GB memory, 500m CPU  
  - Deployment attempted during peak hours (15:00 UTC)  
- **Cascading failures**:  
  - API server CPU spiked to 100%, request queue grew to 10k+  
  - etcd WAL sync latency exceeded 5s, triggering leader elections  
  - Scheduler unable to keep up with pod scheduling demands  
  - Worker nodes hit resource limits before placement decisions  
- **Observable symptoms**:  
  - `kubectl` commands timed out after 30s  
  - CloudWatch showed 100% API server CPU utilization  
  - Pod status showed `Pending` for 400+ pods  
  - Node kubelets overwhelmed with pod creation requests  

---

## Diagnosis Steps  

### 1. Check API server health:
```sh
kubectl get --raw /readyz 2>&1 | head -5
# Output: timeout after 30 seconds
```
