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
