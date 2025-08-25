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
