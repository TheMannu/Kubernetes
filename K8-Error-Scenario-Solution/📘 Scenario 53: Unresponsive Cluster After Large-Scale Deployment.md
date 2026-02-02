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

### 2. Monitor control plane metrics:
```sh
# Via Azure Monitor (since kubectl unavailable)
az monitor metrics list --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ContainerService/managedClusters/{cluster} --metric "apiserver_current_inflight_requests"
# Showed 15k+ inflight requests
```

### 3. Check etcd status:
```sh
# Direct etcdctl access (if possible)
etcdctl endpoint status --cluster
# High latency and leader changes
```

### 4. Analyze deployment patterns:
```sh
# After recovery, check deployment history
kubectl get events --sort-by=.lastTimestamp | grep -i "deployment\|scale" | tail -20
# Showed 500 pod creation events within 60s
```

---

## Root Cause  
**Thundering herd deployment**:  
1. 500 simultaneous pod creation requests to API server  
2. etcd overwhelmed with write operations  
3. No rate limiting in CI/CD pipeline  
4. Missing deployment staging or canary strategy  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Scale down deployment via direct API (if accessible)
curl -k -X PATCH https://${API_SERVER}/apis/apps/v1/namespaces/production/deployments/massive-deploy \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  -H "Content-Type: application/strategic-merge-patch+json" \
  -d '{"spec":{"replicas":10}}'

# 2. Restart control plane components (requires Azure support)
az aks maintenanceconfiguration add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name controlplane-restart \
  --config-file ./restart.json

# 3. Clear stuck pods
kubectl delete pods --field-selector=status.phase=Pending --grace-period=0 --force
```

### Long-term Solution:
```yaml
# RollingUpdate strategy with controlled deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: controlled-deploy
spec:
  replicas: 500
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 10%  # 50 pods at a time
      maxUnavailable: 5%  # 25 pods unavailable during update
  minReadySeconds: 30  # Wait 30s between batches
  progressDeadlineSeconds: 600  # Fail after 10 minutes
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "256Mi"  # Reduced from 1Gi
            cpu: "100m"
```

---

## Lessons Learned  
⚠️ **Batch size matters**: API server has hard limits on concurrent operations  
⚠️ **etcd is the bottleneck**: Writes don't scale linearly with cluster size  
⚠️ **Resource requests compound**: 500 pods × 1GB = 500GB instant demand  

---

## Prevention Framework  

### 1. Deployment Rate Limiting
```yaml
# CI/CD pipeline configuration with rate limits
stages:
- deploy:
    strategy:
      max_parallel: 5  # Max 5 pods simultaneously
      batch_size: 10   # 10 pods per batch
      wait_time: 30    # Wait 30s between batches
    script:
      - kubectl apply -f deployment.yaml --server-side --dry-run=server
      - for i in {1..50}; do
          kubectl scale --replicas=$((i*10)) deployment/massive-deploy
          sleep 30
        done
```

### 2. API Server Protection
```yaml
# API server configuration for burst protection
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --max-requests-inflight=800
    - --max-mutating-requests-inflight=400
    - --target-ram-mb=8192
    - --etcd-servers-overrides=/events#http://etcd-client:2379  # Separate event etcd
```

### 3. Monitoring & Alerting
```yaml
# Prometheus alerts for deployment storms
- alert: APIServerOverload
  expr: rate(apiserver_request_total[5m]) > 5000
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "API server receiving >5k requests/min"

- alert: DeploymentStormDetected
  expr: increase(deployment_controller_syncs_total[1m]) > 100
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "High deployment activity detected"

- alert: etcdHighLatency
  expr: histogram_quantile(0.95, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.5
  for: 2m
  labels:
    severity: critical
```

### 4. Capacity Planning Framework
```sh
# Pre-deployment capacity validation
validate_deployment_capacity() {
  local pods=$1
  local cpu_per_pod=$2
  local mem_per_pod=$3
  
  # Calculate required resources
  local total_cpu=$(echo "$pods * $cpu_per_pod" | bc)
  local total_mem=$(echo "$pods * $mem_per_pod" | bc)
  
  # Check available cluster capacity
  local available_cpu=$(kubectl get nodes -o json | jq '.items[].status.allocatable.cpu' | sed 's/[^0-9.]//g' | awk '{sum+=$1} END {print sum}')
  local available_mem=$(kubectl get nodes -o json | jq '.items[].status.allocatable.memory' | sed 's/[^0-9.]//g' | awk '{sum+=$1} END {print sum/1024/1024/1024}')  # Convert to GB
  
  if (( $(echo "$total_cpu > $available_cpu * 0.7" | bc -l) )); then
    echo "ERROR: CPU request ($total_cpu) exceeds 70% of available capacity ($available_cpu)"
    exit 1
  fi
}
```

---

**Key Deployment Constraints**:  
- **API Server**: ~500-1000 concurrent requests maximum  
- **etcd**: ~50-100 writes per second sustainable  
- **Scheduler**: ~100 pods per second scheduling rate  
- **Kubelet**: ~50 pods per minute creation capacity  

**Scaling Strategy Matrix**:  
```markdown
| Deployment Size | Recommended Strategy          | Batch Size | Interval |
|-----------------|-------------------------------|------------|----------|
| 1-50 pods       | Single batch                  | All        | Immediate|
| 50-200 pods     | RollingUpdate (maxSurge: 20%) | 10-20      | 30s      |
| 200-500 pods    | Phased deployment             | 25         | 60s      |
| 500+ pods       | Multi-stage rollout           | 50         | 120s     |
```

**Debugging Tools**:  
```sh
# Check API server queue depth (after recovery)
kubectl get --raw /metrics | grep apiserver_current_inflight_requests
