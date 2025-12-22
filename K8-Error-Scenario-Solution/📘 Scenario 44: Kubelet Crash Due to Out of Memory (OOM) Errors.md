# 📘 Scenario 44: Kubelet Crash Due to Out of Memory (OOM) Errors

**Category**: Node Resource Management  
**Environment**: Kubernetes 1.20, Bare Metal, 32GB RAM Nodes  
**Impact**: Complete node failure requiring manual intervention  

---

## Scenario Summary  
Kubelet process crashed due to memory exhaustion from unconstrained pod memory consumption, causing node unresponsiveness and requiring manual recovery.

---

## What Happened  
- **Memory pressure buildup**:  
  - Multiple pods without memory limits deployed to single node  
  - Memory consumption grew until system OOM killer terminated kubelet  
- **Failure progression**:  
  - Kubelet logs showed `out of memory` warnings  
  - Node status changed to `NotReady` after kubelet termination  
  - System logs recorded `oom-killer` killing kubelet process (PID 1)  
- **Cascading effects**:  
  - Pods unable to be rescheduled (node still considered allocated)  
  - Critical daemonsets (CNI, monitoring) stopped functioning  
  - Node required hard reboot to recover  

---

## Diagnosis Steps  

### 1. Check system logs for OOM events:
```sh
journalctl -k | grep -i "killed process.*kubelet\|out of memory"
# Output: "Out of memory: Killed process 1234 (kubelet)"
```

### 2. Analyze memory usage before crash:
```sh
kubectl describe node <node> | grep -A10 "Allocated resources"
# Showed 30GB/32GB memory allocated, but actual usage unknown
```

### 3. Check pod resource configurations:
```sh
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].resources.limits.memory == null) | .metadata.namespace + "/" + .metadata.name'
# Showed 15+ pods without memory limits
```

### 4. Review kubelet configuration:
```sh
cat /var/lib/kubelet/config.yaml | grep -A5 systemReserved
# Showed no system memory reservation
```

---
