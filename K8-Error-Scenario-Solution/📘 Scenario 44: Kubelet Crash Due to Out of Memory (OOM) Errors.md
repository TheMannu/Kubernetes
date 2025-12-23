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

## Root Cause  
**Unmanaged memory consumption**:  
1. Pods deployed without memory limits  
2. No kubelet system memory reservations  
3. Missing node-level memory pressure monitoring  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Hard reboot affected node (if unresponsive)
sudo reboot

# 2. After reboot, drain node to prevent immediate rescheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# 3. Apply memory limits to offending deployments
kubectl patch deployment memory-hog -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "app",
          "resources": {
            "limits": {"memory": "512Mi"},
            "requests": {"memory": "256Mi"}
          }
        }]
      }
    }
  }
}'
```

### Long-term Solution:
```yaml
# Kubelet configuration with system reservations
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
systemReserved:
  memory: "2Gi"
  cpu: "500m"
kubeReserved:
  memory: "1Gi"
  cpu: "250m"
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "10%"
```

---

## Lessons Learned  
⚠️ **Kubelet is a system process**: Requires protected memory allocation  
⚠️ **Unlimited pods are dangerous**: Can consume all node resources  
⚠️ **OOM kills are destructive**: Terminate processes without graceful shutdown  

---

## Prevention Framework  

### 1. Resource Policy Enforcement
```yaml
# OPA/Gatekeeper constraint for memory limits
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceLimits
metadata:
  name: require-memory-limits
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet", "DaemonSet"]
  parameters:
    limits:
      memory: "required"
    requests:
      memory: "required"
```

### 2. Kubelet Configuration Hardening
```yaml
# Kubelet systemd unit with memory protection
[Service]
MemoryHigh=4G
MemoryMax=4.5G
MemorySwapMax=0  # Disable swap for kubelet
```

### 3. Monitoring & Alerting
```yaml
# Prometheus alerts for memory pressure
- alert: NodeMemoryPressure
  expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.85
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Node {{ $labels.instance }} memory usage >85%"

- alert: PodWithoutMemoryLimits
  expr: count(kube_pod_container_info{container="",container_memory_limit_bytes="0"}) by (namespace,pod) > 0
  for: 1h
  labels:
    severity: warning
```
