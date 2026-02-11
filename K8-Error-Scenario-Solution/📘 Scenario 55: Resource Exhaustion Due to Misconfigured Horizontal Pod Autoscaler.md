# 📘 Scenario 55: Resource Exhaustion Due to Misconfigured Horizontal Pod Autoscaler

**Category**: Autoscaling & Resource Management  
**Environment**: Kubernetes 1.22, AWS EKS, 100-node cluster  
**Impact**: Cluster scaled to 150% capacity, 50% cost overrun, performance degradation  

---

## Scenario Summary  
An overly aggressive HPA configuration triggered runaway scaling, exhausting cluster resources and driving up cloud costs while actually degrading application performance due to resource contention.

---

## What Happened  
- **Triggering event**:  
  - Legitimate traffic spike increased CPU utilization from 40% to 85%  
  - HPA with CPU target of 50% triggered immediate scaling  
- **Runaway scaling**:  
  - Pods scaled from 10 → 100 in 15 minutes  
  - Each new pod increased overall CPU load (cold start overhead)  
  - Scaling continued despite plateauing external traffic  
- **Observed symptoms**:  
  - Node pool scaled from 50 → 150 nodes ($300/hr cost)  
  - Application latency increased due to resource contention  
  - etcd performance degraded from 1000+ pods per node  
  - Cloud bill alerts triggered at 150% budget threshold  

---

## Diagnosis Steps  

### 1. Check HPA scaling events:
```sh
kubectl describe hpa webapp -n production
# Output:
# CurrentReplicas: 100, DesiredReplicas: 100
# Scaling events: 90 scaling events in last 15 minutes
```

### 2. Analyze metrics trends:
```sh
# Query Prometheus for CPU utilization during incident
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod) / 
sum(kube_pod_container_resource_requests{namespace="production",resource="cpu"}) by (pod)
# Showed CPU spike followed by sustained high utilization despite scaling
```

### 3. Check node resource pressure:
```sh
kubectl top nodes
# Output: All nodes at 80%+ CPU, memory pressure warnings
```

### 4. Review HPA configuration:
```sh
kubectl get hpa webapp -n production -o yaml | yq '.spec'
# Showed:
# metrics:
# - type: Resource
#   resource:
#     name: cpu
#     target:
#       type: Utilization
#       averageUtilization: 50  # Too aggressive
# minReplicas: 1
# maxReplicas: 100  # Too high without safeguards
```

---

## Root Cause  
**Single-metric autoscaling failure**:  
1. HPA using only CPU metric without stabilization window  
2. `maxReplicas` set too high without pod disruption considerations  
3. No scaling cooldown between scaling events  
4. Missing memory-based scaling limits  

---

## Fix/Workaround  

### Emergency Stabilization:
```sh
# 1. Temporarily cap HPA maximum replicas
kubectl patch hpa webapp -n production -p '{"spec":{"maxReplicas":20}}'

# 2. Manually scale down overloaded deployment
kubectl scale deployment webapp -n production --replicas=20

# 3. Suspend cluster autoscaler temporarily
kubectl scale deployment cluster-autoscaler -n kube-system --replicas=0
```

### Long-term Solution:
```yaml
# Multi-metric HPA with stabilization windows
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
  maxReplicas: 50  # Realistic maximum
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60  # Max 2 pods per minute when scaling down
    scaleUp:
      stabilizationWindowSeconds: 60  # Wait 1 minute before scaling up
      policies:
      - type: Pods
        value: 5
        periodSeconds: 60  # Max 5 pods per minute when scaling up
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # More realistic target
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
⚠️ **Scaling too fast creates feedback loops**: New pods consume resources during initialization  
⚠️ **Single metrics are dangerous**: Must combine CPU, memory, and custom metrics  
⚠️ **Stabilization windows are critical**: Prevent rapid oscillation  
⚠️ **Cost controls matter**: Unlimited scaling equals unlimited costs  

---

## Prevention Framework  

### 1. HPA Configuration Validation
```sh
# Pre-apply HPA validation
validate_hpa_config() {
  local hpa_file=$1
  
  local max_replicas=$(yq '.spec.maxReplicas' $hpa_file)
  local metrics_count=$(yq '.spec.metrics | length' $hpa_file)
  
  # Enforce maximum scaling limits
  if [ $max_replicas -gt 100 ]; then
    echo "ERROR: maxReplicas ($max_replicas) exceeds safety limit of 100"
    exit 1
  fi
  
  # Require multiple metrics
  if [ $metrics_count -lt 2 ]; then
    echo "WARNING: HPA configured with only $metrics_count metric(s)"
  fi
  
  # Check for stabilization windows
  if ! yq '.spec.behavior.scaleUp.stabilizationWindowSeconds' $hpa_file > /dev/null 2>&1; then
    echo "WARNING: No stabilization window configured"
  fi
}
```

### 2. Cost Control Integration
```yaml
# Custom controller for cost-aware scaling
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cost-aware-scaler
spec:
  template:
    spec:
      containers:
      - name: scaler
        image: cost-aware-scaler:1.0
        env:
        - name: MAX_COST_PER_HOUR
          value: "100"  # $100/hour maximum
        - name: CLOUD_PROVIDER
          value: "aws"
        - name: REGION
          value: "us-west-2"
```

### 3. Monitoring & Alerting
```yaml
# Prometheus alerts for runaway scaling
- alert: HPAExcessiveScaling
  expr: increase(hpa_scaling_total{direction="up"}[10m]) > 20
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "HPA scaling up too rapidly ({{ $value }} events in 10m)"

- alert: HPAAtMaxReplicas
  expr: kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_max_replicas
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} at maximum replicas for 15+ minutes"

- alert: ClusterCostExceeded
  expr: aws_ec2_running_instances * 0.10 > 100  # Assuming $0.10/hr per instance
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Cluster cost exceeds $100/hour threshold"
```

### 4. Load Testing Framework
```yaml
# Automated scaling test before production deployment
apiVersion: batch/v1
kind: Job
metadata:
  name: hpa-load-test
spec:
  template:
    spec:
      containers:
      - name: loader
        image: loadimpact/k6
        command: ["k6"]
        args:
        - "run"
        - "--vus"
        - "1000"
        - "--duration"
        - "10m"
        - "/scripts/load-test.js"
        env:
        - name: TARGET_URL
          value: "http://webapp.production.svc.cluster.local"
        volumeMounts:
        - name: scripts
          mountPath: /scripts
      volumes:
      - name: scripts
        configMap:
          name: load-test-scripts
```

---

**Key HPA Design Principles**:  
- **Multiple Metrics**: Combine CPU, memory, and application-specific metrics  
- **Stabilization Windows**: Prevent rapid oscillation (scaleUp: 60-120s, scaleDown: 300-600s)  
- **Gradual Scaling**: Limit pods per minute scaling rates  
- **Cost Awareness**: Integrate with cloud billing APIs  
- **Capacity Planning**: Realistic `maxReplicas` based on actual capacity  
