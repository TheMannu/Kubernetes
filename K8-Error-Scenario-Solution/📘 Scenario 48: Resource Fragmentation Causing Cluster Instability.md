# 📘 Scenario 48: Resource Fragmentation Causing Cluster Instability

**Category**: Cluster Scheduling & Resource Optimization  
**Environment**: Kubernetes 1.23, Bare Metal with Heterogeneous Nodes  
**Impact**: Performance degradation, increased OOM kills, and failed deployments due to uneven resource distribution  

---

## Scenario Summary  
Resource fragmentation from imbalanced pod scheduling led to "hot" nodes experiencing performance issues while "cold" nodes remained underutilized, causing cluster-wide inefficiency and reliability problems.

---

## What Happened  
- **Imbalanced scheduling over time**:  
  - Scheduler favored nodes with existing resource allocations (bin-packing algorithm)  
  - 80% of pods concentrated on 30% of nodes  
  - Node 1: 45 pods, 95% CPU utilization  
  - Node 2: 8 pods, 15% CPU utilization  
- **Observed symptoms**:  
  - High-latency API responses from overloaded nodes  
  - Frequent OOM kills on "hot" nodes  
  - Failed pod scheduling: `0/1 nodes available: Insufficient cpu` despite overall cluster capacity  
  - Uneven wear on hardware components  
- **Root analysis**:  
  - No pod anti-affinity rules for same-application pods  
  - Missing topology spread constraints  
  - Default scheduler scoring favored packing over distribution  

---

## Diagnosis Steps  

### 1. Check node resource distribution:
```sh
kubectl get nodes -o custom-columns="NAME:.metadata.name,CPU-REQ:status.allocatable.cpu,CPU-USE:status.allocatable.cpu - .status.allocatable.cpu,CPU%:.status.allocatable.cpu - .status.allocatable.cpu / .status.allocatable.cpu * 100,PODS:status.allocatable.pods - .status.allocatable.pods"
# Showed 80% CPU utilization on 3/10 nodes
```

### 2. Analyze pod distribution:
```sh
kubectl get pods -A -o wide | awk '{print $8}' | sort | uniq -c | sort -nr
# Output: node-1: 45 pods, node-2: 8 pods, node-3: 42 pods
```

### 3. Check scheduling failures:
```sh
kubectl get events -A --sort-by=.lastTimestamp | grep -i "insufficient\|failed scheduling"
# Multiple "Insufficient cpu" errors despite cluster having capacity
```

### 4. Review node resource fragmentation:
```sh
kubectl describe node <overloaded-node> | grep -A20 "Allocated resources"
# Showed many small resource allocations causing fragmentation
```

---

## Root Cause  
**Scheduler algorithm limitations**:  
1. Default bin-packing optimization favored node saturation  
2. No topology spread constraints for workload distribution  
3. Missing pod anti-affinity for same-service instances  
4. Heterogeneous node sizes exacerbated fragmentation  

---

## Fix/Workaround  

### Immediate Remediation:
```sh
# 1. Drain overloaded nodes (carefully, with PDBs)
kubectl drain <overloaded-node> --ignore-daemonsets --delete-emptydir-data

# 2. Apply topology spread constraints
kubectl patch deployment critical-app -p '{
  "spec": {
    "template": {
      "spec": {
        "topologySpreadConstraints": [{
          "maxSkew": 1,
          "topologyKey": "kubernetes.io/hostname",
          "whenUnsatisfiable": "ScheduleAnyway",
          "labelSelector": {"matchLabels": {"app": "critical-app"}}
        }]
      }
    }
  }
}'
```

### Long-term Solution:
```yaml
# Deployment with balanced scheduling
apiVersion: apps/v1
kind: Deployment
metadata:
  name: balanced-app
spec:
  replicas: 10
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: balanced-app
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["balanced-app"]
              topologyKey: kubernetes.io/hostname
```

---

## Lessons Learned  
⚠️ **Bin-packing creates hotspots**: Default scheduler favors density over distribution  
⚠️ **Topology matters**: Physical node distribution affects performance  
⚠️ **Fragmentation is cumulative**: Small inefficiencies compound over time  

---

## Prevention Framework  

### 1. Scheduler Configuration
```yaml
# Kube-scheduler configuration profile
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated  # Changed from MostAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
  - name: NodeResourcesBalancedAllocation
    args:
      resources:
      - name: cpu
        weight: 1
      - name: memory
        weight: 1
```

### 2. Resource Usage Monitoring
```yaml
# Prometheus alerts for resource fragmentation
- alert: NodeResourceImbalance
  expr: stddev(kube_node_status_allocatable_cpu_cores - kube_node_status_allocatable_cpu_cores) > kube_node_status_allocatable_cpu_cores * 0.3
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Uneven CPU allocation across nodes (stddev >30%)"

- alert: PodDistributionSkew
  expr: max(kube_pod_info) by (node) / avg(kube_pod_info) by () > 2
  for: 1h
  labels:
    severity: warning
```

### 3. Automated Rebalancing
```yaml
# Descheduler CronJob for periodic rebalancing
apiVersion: batch/v1
kind: CronJob
metadata:
  name: descheduler
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: descheduler-sa
          containers:
          - name: descheduler
            image: registry.k8s.io/descheduler/descheduler:v0.26.0
            command: ["/bin/descheduler"]
            args:
            - "--policy-config-file=/policy-dir/policy.yaml"
            - "--v=3"
            volumeMounts:
            - mountPath: /policy-dir
              name: policy-volume
          volumes:
          - name: policy-volume
            configMap:
              name: descheduler-policy
```

### 4. CI/CD Scheduling Policies
```yaml
# Kustomize patch for all deployments
apiVersion: apps/v1
kind: Deployment
metadata:
  name: not-important
spec:
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: placeholder
```

---

**Key Scheduling Strategies**:  
- **Topology Spread Constraints**: Distribute pods across failure domains  
- **Pod Anti-affinity**: Prevent co-location of same-service pods  
- **Node Affinity**: Control placement based on node characteristics  
- **Resource Limits**: Prevent single pods from dominating nodes  

**Debugging Tools**:  
```sh
# Visualize node resource allocation
kubectl describe nodes | grep -A10 "Allocated resources" | grep -E "(cpu|memory|pods)"

# Check scheduler scoring
kubectl get events -A | grep -i "scheduled" | tail -20

# Simulate scheduling decisions
kubectl create deployment test-sched --image=nginx --replicas=5 --dry-run=client -o yaml | \
  kubectl apply -f - && sleep 5 && kubectl get pods -o wide

# Analyze fragmentation patterns
kubectl get pods -A -o json | jq -r '.items[] | .spec.nodeName' | sort | uniq -c
```

**Resource Fragmentation Prevention Matrix**:  
```markdown
| Problem                     | Solution                          | Implementation                  |
|-----------------------------|-----------------------------------|----------------------------------|