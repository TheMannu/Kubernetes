# 📘 Scenario 26: Taints and Tolerations Mismatch Prevented Workload Scheduling

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.22, AKS GPU Node Pool  
**Impact**: GPU-accelerated workloads failed to schedule for 6+ hours  

---

## Scenario Summary  
New GPU nodes remained underutilized because critical workloads lacked required tolerations, causing scheduling deadlocks and resource starvation for AI/ML pipelines.

---

## What Happened  
- **Infrastructure upgrade**:  
  - New `NCv3` node pool added with `node-role.kubernetes.io/gpu:NoSchedule`  
  - Node autoscaler provisioned 8 GPU nodes ($12/hr each)  
- **Scheduling failures**:  
  - `kubectl get events` showed `0/8 nodes available: 8 node(s) had untolerated taint`  
  - GPU utilization metrics showed 0% usage  
- **Diagnosis findings**:  
  - 47 deployments lacked GPU tolerations  
  - Node selector `accelerator: nvidia-tesla` still pointed to old nodes  

---

## Diagnosis Steps  

### 1. Check pending pods:
```sh
kubectl get pods -A --field-selector status.phase=Pending -o wide | grep gpu
```

### 2. Inspect scheduling barriers:
```sh
kubectl describe pod <pending-pod> | grep -A10 Events
# Showed "node(s) had untolerated taint {node-role.kubernetes.io/gpu: }"
```
