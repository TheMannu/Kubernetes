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
