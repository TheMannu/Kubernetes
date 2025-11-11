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
