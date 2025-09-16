# 📘 Scenario 19: Multiple Nodes Marked Unreachable Due to Flaky Network Interface

**Category**: Infrastructure Reliability  
**Environment**: Kubernetes v1.22, Bare-metal, Bonded NICs  
**Impact**: Intermittent node failures, pod evictions, and workload disruptions  

---

## Scenario Summary  
A flapping network interface caused multiple nodes to oscillate between `Ready` and `NotReady` states, triggering unnecessary pod evictions and workload rescheduling.

---

## What Happened  
- **Network instability**:  
  - Nodes randomly reported `NotReady` for 30-60 second intervals  
  - `kube-controller-manager` logs showed frequent `NodeNotReady` events
- **Observed symptoms**:  
  - Pods evicted with `NodeNetworkUnavailable` reason  
  - Cluster autoscaler provisioned unnecessary replacement nodes  
  - Storage systems (CSI) timed out during network drops  
- **Root discovery**:  
  - Switch logs showed `port up/down` events every 2-3 minutes  
  - `ethtool` reported `link flaps: 142` on affected nodes  

---