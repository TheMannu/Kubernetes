# 📘 Scenario 40: Cluster-Wide Crash Due to Misconfigured Resource Quotas

**Category**: Cluster Resource Management  
**Environment**: Kubernetes 1.24, Multi-tenant Cluster  
**Impact**: Complete deployment freeze affecting 200+ namespaces  

---

## Scenario Summary  
Overly restrictive resource quotas applied cluster-wide prevented all new pod scheduling, causing a complete halt to deployments, scaling operations, and pod replacements.

---

## What Happened  
- **Quota policy change**:  
  - New `ResourceQuota` applied to all namespaces via `Namespace` selector  
  - CPU limits set to `100m` and memory to `256Mi` per namespace (far below existing usage)  
- **Immediate failures**:  
  - All new pods failed with `Failed quota: exceeded quota`  
  - Horizontal Pod Autoscaler attempts triggered quota violations  
  - Crash-looping pods couldn't restart (exceeded quota)  
- **Business impact**:  
  - Zero-downtime deployments failed mid-rollout  
  - Production services degraded as pods terminated  
  - Emergency fixes blocked by quota restrictions  

---

## Diagnosis Steps  

### 1. Identify scheduling failures:
```sh
kubectl get events -A --sort-by=.lastTimestamp | grep -i "quota\|forbidden"
# Showed "exceeded quota" errors across namespaces
```

### 2. Check quota usage:
```sh
kubectl get quota -A
# Output showed 100% utilization across all namespaces
```
