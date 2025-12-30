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

## Diagnosis Steps  

### 1. Check HPA status:
```sh
kubectl get hpa -n production
# Output: 
# NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# webapp  Deployment/webapp  90%/80%   2         10        2          15d
```

### 2. Inspect HPA events:
```sh
kubectl describe hpa webapp -n production | grep -A10 Events
# Showed no scaling events despite high utilization
```

### 3. Analyze pod resource configuration:
```sh
kubectl get deployment webapp -n production -o yaml | yq '.spec.template.spec.containers[].resources'
# Output:
# limits:
#   cpu: 100m
#   memory: 128Mi
# requests:
#   cpu: 50m
#   memory: 64Mi
```

### 4. Check actual resource usage:
```sh
kubectl top pods -n production --containers
# Output:
# webapp-abc123    app    95m (95%)   56Mi (43%)
```

---

## Root Cause  
**Resource configuration mismatch**:  
1. HPA uses `requests` for utilization calculation, not actual resource usage  
2. Pod limits set too low, causing 100% utilization at normal load  
3. No buffer between normal operations and scaling thresholds  

---

## Fix/Workaround  

### Immediate Scaling Intervention:
```sh
# 1. Manually scale deployment (emergency)
kubectl scale deployment webapp -n production --replicas=5

# 2. Update resource limits
kubectl patch deployment webapp -n production -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "app",
          "resources": {
            "limits": {"cpu": "500m", "memory": "256Mi"},
            "requests": {"cpu": "200m", "memory": "128Mi"}
          }
        }]
      }
    }
  }
}'

# 3. Verify HPA recalculates
kubectl get hpa webapp -n production -w
```

### Long-term Solution:
```yaml
# HPA with multiple metrics and appropriate targets
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 100
```

---

## Lessons Learned  
⚠️ **HPA math matters**: Utilization = (Actual Usage / Requests) × 100%  
⚠️ **Buffer capacity required**: Normal ops should be at 40-60% of requests  
⚠️ **Multiple metrics prevent single-point failures**: CPU alone isn't sufficient  

---

## Prevention Framework  

### 1. HPA Configuration Validation
```sh
# Pre-deployment HPA validation script
validate_hpa_config() {
  local deployment=$1
  local namespace=$2
  
  # Get resource requests
  local requests_cpu=$(kubectl get deployment $deployment -n $namespace -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')
  
  # Check if requests are sufficient for scaling calculations
  if [[ "$requests_cpu" =~ m$ ]]; then
    local cpu_milli=${requests_cpu%m}
    if [ $cpu_milli -lt 100 ]; then
      echo "ERROR: CPU requests ($requests_cpu) too low for effective HPA scaling"
      exit 1
    fi
  fi
}
```

### 2. Resource Sizing Guidelines
```yaml
# Resource sizing policy template
resources:
  requests:
    cpu: "200m"     # 40-60% of expected average usage
    memory: "256Mi"  # 50-70% of expected average usage
  limits:
    cpu: "500m"     # 2.5x requests for burst capacity
    memory: "512Mi" # 2x requests for memory safety
```

### 3. Monitoring & Alerting
```yaml
# Prometheus alerts for HPA health
- alert: HPAStuckAtMin
  expr: kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_min_replicas
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} stuck at minimum replicas"
