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