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

### Gradual Quota Application:
```yaml
# Quota with buffer for existing workloads
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    pods: "50"
```

---

## Lessons Learned  
⚠️ **Quotas are enforcement boundaries**: Apply cautiously to avoid breaking existing workloads  
⚠️ **Current usage matters**: Must baseline before setting limits  
⚠️ **Dry-run is essential**: Test quota impact before enforcement  

---

## Prevention Framework  

### 1. Quota Validation Workflow
```sh
# Pre-apply quota validation script
validate_quota() {
  local namespace=$1
  local quota_file=$2
  
  # Get current usage
  local current_cpu=$(kubectl get quota -n $namespace -o json 2>/dev/null | jq '.items[].status.used."limits.cpu" // "0"' | sed 's/"//g')
  local quota_cpu=$(yq '.spec.hard."limits.cpu"' $quota_file)
  
  if [ "$current_cpu" -gt "$quota_cpu" ]; then
    echo "ERROR: Current CPU usage ($current_cpu) exceeds proposed quota ($quota_cpu)"
    exit 1
  fi
}
```

### 2. Admission Control Safeguards
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sValidResourceQuota
metadata:
  name: quota-change-validation
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["ResourceQuota"]
  parameters:
    maxReductionPercent: 20  # Cannot reduce quota by more than 20% at once
    requiredLabels:
    - change-request-id
```

### 3. Monitoring & Alerts
```yaml
# Prometheus alerts
- alert: QuotaExceeded
  expr: kube_resourcequota_status_used > kube_resourcequota * 0.9
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Namespace {{ $labels.namespace }} approaching quota limit"

- alert: QuotaChangeImpact
  expr: abs(delta(kube_resourcequota_status_used[1h])) > kube_resourcequota_status_used * 0.5
  labels:
    severity: critical
  annotations:
    summary: "Quota change causing major usage shift"
```

### 4. Change Management Process
```markdown
## Quota Change Protocol
1. **Baseline current usage**:
   ```sh
   kubectl describe quota -n <namespace>
   ```
2. **Calculate safe limits** (current usage + 30% buffer)  
3. **Dry-run simulation**:
   ```sh
   kubectl apply -f new-quota.yaml --dry-run=server
   ```
4. **Gradual rollout** using canary namespaces  
5. **Monitor impact** for 24 hours before full rollout  
```

---

**Key Quota Metrics to Monitor**:  
- `kube_resourcequota` - defined quota limits  
- `kube_resourcequota_status_used` - current usage  
- `kube_pod_resource_requests` - requested resources  
- `kube_pod_resource_limits` - limit resources  

**Debugging Tools**:  
```sh
# Check quota usage across all namespaces
kubectl get quota -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,CPU-REQ:.status.used.requests.cpu,CPU-LIM:.status.used.limits.cpu,MEM-REQ:.status.used.requests.memory,MEM-LIM:.status.used.limits.memory"
