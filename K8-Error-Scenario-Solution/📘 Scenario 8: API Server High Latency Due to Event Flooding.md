📘 Scenario 8: API Server High Latency Due to Event Flooding

**Category**: API Server Performance  
**Environment**: Kubernetes v1.23, Azure AKS  
**Impact**: Cluster-wide API latency spikes, scheduler disruptions  

---

## Scenario Summary  
A malfunctioning controller flooded the cluster with 50+ events/second, overwhelming etcd and causing API server latency to exceed 5s.

---

## What Happened  
- **API degradation**:  
  - `kubectl` commands timed out (`Error from server (Timeout)`)  
  - Scheduler could not update pod status (`leader election lost`)  
- **Event storm**:  
  - 500,000+ events in default namespace  
  - etcd WAL growth rate spiked to 50MB/minute  
- **Controller misbehavior**:  
  - Continuous `FailedCreate` events for non-critical conditions  

---

## Diagnosis Steps  

### 1. Identify event source:
```sh
kubectl get events --all-namespaces --sort-by='.metadata.creationTimestamp' | \
  awk '{print $2}' | sort | uniq -c | sort -n
```

### 2. Check etcd metrics:
```sh
# Via Prometheus
etcd_disk_wal_fsync_duration_seconds{quantile="0.99"} > 1
etcd_server_quota_backend_bytes{cluster="true"} > 90%
```

### 3. Locate problematic controller:
```sh
kubectl logs -n <namespace> <controller-pod> | \
  grep -i "event.*failed" -m 20
```

### 4. Verify API server health:
```sh
kubectl get --raw='/metrics' | \
  grep -E 'apiserver_request_duration_seconds|apiserver_current_inflight_requests'
```

---

## Root Cause  
**Uncontrolled event generation**:  
1. **No rate limiting**: Controller logged identical errors in tight loop  
2. **No deduplication**: Same event recorded hundreds of times  
3. **No cleanup**: Default event TTL of 1h insufficient  

---

## Fix/Workaround  

### Immediate Actions:
```sh
# Bulk delete events (carefully!)
kubectl delete events --all --namespace=default

# Scale down offending controller
kubectl scale deploy <controller> --replicas=0
```

### Controller Patch Example:
```go
// Before: Uncontrolled events
recorder.Eventf(obj, "Warning", "FailedCreate", "Error creating resource")

// After: Rate-limited with deduplication
if time.Since(lastEvent) > 5*time.Minute {
    recorder.Eventf(obj, "Warning", "FailedCreate", 
        "Error creating resource (repeated %d times)", errorCount)
    lastEvent = time.Now()
    errorCount = 0
}
```

---

## Lessons Learned  
⚠️ **Events are etcd writes**: Each event consumes etcd I/O capacity  
⚠️ **Cascading failures**: API latency affects all controllers/schedulers  

---

## Prevention Framework  

### 1. Cluster Configuration
```yaml
# API server flags
--event-ttl=30m  # Reduce from default 1h
--enable-admission-plugins=EventRateLimit
--admission-control-config-file=eventconfig.yaml
```

### 2. Event Rate Limiting Policy
```yaml
# eventconfig.yaml
apiVersion: eventratelimit.admission.k8s.io/v1alpha1
kind: Configuration
limits:
  - type: Namespace
    qps: 50
    burst: 100
```