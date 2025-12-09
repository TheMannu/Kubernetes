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

### 3. Analyze quota definitions:
```sh
kubectl get quota -o yaml | yq '.spec.hard'
# Revealed restrictive limits: cpu: "100m", memory: "256Mi"
```

### 4. Compare with actual usage:
```sh
kubectl top pods -A --no-headers | awk '{cpu+=$2; mem+=$3} END {print "CPU: " cpu "m, Memory: " mem "Mi"}'
# Showed usage far exceeding new quota limits
```

---

## Root Cause  
**Aggressive quota enforcement without validation**:  
1. Quotas applied without assessing current resource consumption  
2. No dry-run or staged rollout  
3. Missing monitoring for quota utilization trends  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Temporarily remove restrictive quotas
kubectl delete quota -A --all

# 2. Restart critical deployments
kubectl rollout restart deployment -n production

# 3. Apply corrected quotas (after analysis)
kubectl apply -f corrected-quotas.yaml
```
