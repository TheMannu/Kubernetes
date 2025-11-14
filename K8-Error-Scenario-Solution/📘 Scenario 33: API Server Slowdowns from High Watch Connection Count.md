# 📘 Scenario 33: API Server Slowdowns from High Watch Connection Count

**Category**: API Server Performance  
**Environment**: Kubernetes 1.23, OpenShift 4.10  
**Impact**: API latency increased from 50ms to 5s, affecting all cluster operations  

---

## Scenario Summary  
A misconfigured custom controller opened thousands of persistent watch connections without proper cleanup, exhausting API server resources and causing cluster-wide performance degradation.

---

## What Happened  
- **Controller misbehavior**:  
  - Custom operator created new watch connections in each reconciliation loop  
  - No connection reuse or cleanup logic implemented  
- **Performance symptoms**:  
  - `kubectl` commands timed out with `apiserver request timeout`  
  - API server memory usage grew to 12GB (normal: 2GB)  
  - etcd showed high `watch_stream` backlog  
- **Connection analysis**:  
  - 8,400 active watch connections from single namespace  
  - Single pod maintaining 200+ concurrent watches  

---

## Diagnosis Steps  

### 1. Monitor API server metrics:
```sh
kubectl get --raw /metrics | grep -E "apiserver_registered_watchers|apiserver_current_inflight_requests"
# Showed 12,000+ registered watchers (normal: 200-500)
```

### 2. Identify connection sources:
```sh
kubectl get --raw /metrics | grep "apiserver_watch_events_total" | \
  sort -nr -k2 | head -10
```

### 3. Check etcd performance:
```sh
kubectl -n openshift-etcd logs -l app=etcd --tail=100 | grep -i "watch.*backlog"
```

### 4. Inspect client behavior:
```sh
kubectl top pods -n custom-operators | sort -nr -k3 | head -5
# Showed one pod using 800MB+ memory
```

---

## Root Cause  
**Resource leak in watch management**:  
1. Controller created new watches instead of reusing existing ones  
2. No connection timeout or cleanup mechanisms  
3. Missing client-side watch termination on errors  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Scale down offending controller
kubectl -n custom-operators scale deployment/misbehaving-controller --replicas=0

# 2. Force connection cleanup
kubectl -n openshift-kube-apiserver rollout restart deployment/kube-apiserver

# 3. Monitor recovery
watch "kubectl get --raw /metrics | grep apiserver_registered_watchers"
```

### Controller Patch:
```go
// Before: Leaking watches
for {
    watcher, err := client.CoreV1().Pods("").Watch(context.TODO(), opts)
    // Never closed watcher
}

// After: Proper watch management
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Minute)
defer cancel()

watcher, err := client.CoreV1().Pods("").Watch(ctx, opts)
defer watcher.Stop()

for event := range watcher.ResultChan() {
    // Process events
}
```

---


## Lessons Learned  
⚠️ **Watches are expensive**: Each consumes API server resources  
⚠️ **Client-go needs careful usage**: Automatic retries can compound issues  
⚠️ **API server has soft limits**: Performance degrades before hard failures  

---

## Prevention Framework  

### 1. Client Configuration
```go
// Proper client-go setup
config := &rest.Config{
    QPS:           50,
    Burst:         100,
    Timeout:       30 * time.Second,
}

// Shared informer with resync period
informerFactory := informers.NewSharedInformerFactory(clientset, 30*time.Minute)
podInformer := informerFactory.Core().V1().Pods()
```

### 2. Resource Limits
```yaml
# API server flags for protection
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  containers:
  - command:
    - kube-apiserver
    - --max-requests-inflight=800
    - --max-mutating-requests-inflight=400
    - --watch-cache-sizes=100
```

### 3. Monitoring
```yaml
# Critical Prometheus alerts
- alert: APIWatcherLeak
  expr: apiserver_registered_watchers > 1000
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "API server watcher count too high ({{ $value }})"

- alert: APIConnectionSpike
  expr: rate(apiserver_current_inflight_requests[5m]) > 100
  for: 2m
  labels:
    severity: warning
```

### 4. Development Standards
```markdown
## Watch Implementation Checklist
1. **Use SharedInformers** instead of direct watches  
2. **Implement timeouts**:  
   ```go
   ctx, cancel := context.WithTimeout(ctx, 30*time.Minute)
   defer cancel()
   ```
3. **Monitor connection count** in controller metrics  
4. **Test under load** with 1000+ resources  

--

**Key Metrics to Monitor**:  
- `apiserver_registered_watchers`  
- `apiserver_current_inflight_requests`  
- `etcd_watchers`  
- `process_resident_memory_bytes{job="apiserver"}`  

**Debugging Tools**:  
```sh
# Real-time watch monitoring
kubectl get --raw /metrics | grep apiserver_watch

# Connection source analysis
ss -tnp | grep 6443 | awk '{print $6}' | cut -d: -f2 | sort | uniq -c | sort -nr

# Memory profiling
kubectl -n openshift-kube-apiserver exec kube-apiserver -- curl http://localhost:6060/debug/pprof/heap
```

**Watch Management Best Practices**:  
```markdown
1. **Use SharedInformerFactory** for resource caching  
2. **Set reasonable timeouts** (30min max)  
3. **Implement exponential backoff** for reconnects  
4. **Monitor your controller's watch count**  
5. **Test connection cleanup** during graceful shutdown  
```