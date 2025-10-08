# 📘 Scenario 27: Node Bootstrap Failure Due to Unavailable Container Registry

**Category**: Cluster Provisioning  
**Environment**: Kubernetes 1.21, On-prem, Air-gapped Registry  
**Impact**: 100% node provisioning failure during registry outage  

---

## Scenario Summary  
A private container registry outage prevented new nodes from joining the cluster by blocking pulls of critical bootstrap images (`pause`, `kube-proxy`, CNI), causing complete failure of scaling operations.

---

## What Happened  
- **Registry maintenance**:  
  - Storage backend upgrade caused 2-hour registry unavailability  
  - No maintenance window coordination with cluster operations  
- **Bootstrap failures**:  
  - `containerd` logged `failed to pull image: registry.internal:5000/pause:3.4.1`  
  - Nodes stuck in `NotReady` with `ContainerRuntimeNotReady` condition  
- **Cascading effects**:  
  - Cluster autoscaler triggered 15 failed node launches  
  - Emergency manual scaling attempts also failed  

---