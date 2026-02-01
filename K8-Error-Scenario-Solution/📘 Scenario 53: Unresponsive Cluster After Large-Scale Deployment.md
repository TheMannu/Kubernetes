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
