# 📘 Scenario 19: Multiple Nodes Marked Unreachable Due to Flaky Network Interface

**Category**: Infrastructure Reliability  
**Environment**: Kubernetes v1.22, Bare-metal, Bonded NICs  
**Impact**: Intermittent node failures, pod evictions, and workload disruptions  

---

## Scenario Summary  
A flapping network interface caused multiple nodes to oscillate between `Ready` and `NotReady` states, triggering unnecessary pod evictions and workload rescheduling.

---