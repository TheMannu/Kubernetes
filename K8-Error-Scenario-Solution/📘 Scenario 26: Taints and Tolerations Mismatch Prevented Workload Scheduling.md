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

### 3. Verify node taints:
```sh
kubectl get nodes -l accelerator=nvidia-tesla -o json | \
  jq -r '.items[].spec.taints'
# Output: [{"effect":"NoSchedule","key":"node-role.kubernetes.io/gpu"}]
```

### 4. Audit workload specs:
```sh
kubectl get deploy -A -o json | \
  jq -r '.items[] | select(.spec.template.spec.nodeSelector?.accelerator=="nvidia-tesla") | 
  .metadata.namespace + "/" + .metadata.name' | \
  xargs -I{} kubectl get deploy -n {} -o json | \
  jq -r '.spec.template.spec.tolerations' | grep -L "node-role.kubernetes.io/gpu"
```

---

## Root Cause  
**Scheduling policy drift**:  
1. New taints introduced without workload updates  
2. CI/CD pipelines didn't enforce toleration standards  
3. No validation between node selectors and taints  

---

## Fix/Workaround  

### Immediate Resolution:
```yaml
# Patch deployments with tolerations
kubectl patch deploy <name> -p '{
  "spec":{
    "template":{
      "spec":{
        "tolerations":[{
          "key":"node-role.kubernetes.io/gpu",
          "operator":"Exists",
          "effect":"NoSchedule"
        }]
      }
    }
  }
}'
```

### Long-term Solution:
```yaml
# Kustomize patch for GPU workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-workload
spec:
  template:
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
```

---


## Lessons Learned  
⚠️ **Taints break scheduling silently**: No warnings during apply  
⚠️ **Node selectors ≠ taints**: Both must be coordinated  
⚠️ **Costly idle resources**: Untolerated GPU nodes waste $100+/hr  

---
     
## Prevention Framework  

### 1. Admission Control
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredTolerations
metadata:
  name: gpu-toleration-check
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment","DaemonSet"]
    labelSelector:
      matchExpressions:
      - key: accelerator
        operator: In
        values: ["nvidia-tesla"]
  parameters:
    tolerations:
    - key: "node-role.kubernetes.io/gpu"
      operator: "Exists"
```
         
### 2. CI/CD Validation
```sh
# Pre-deployment check
if grep -q "nvidia-tesla" $MANIFEST && ! grep -q "node-role.kubernetes.io/gpu" $MANIFEST; then
  echo "ERROR: GPU workloads require tolerations"
  exit 1
fi
```
                       
### 3. Node Pool Testing
```yaml
# Test deployment template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-test
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        accelerator: nvidia-tesla
      tolerations:
      - key: "node-role.kubernetes.io/gpu"
        operator: "Exists"
      containers:
      - name: stress
        image: nvidia/cuda:11.0-base
        command: ["nvidia-smi"]
```
            
### 4. Monitoring
```yaml
# Prometheus alerts
- alert: UntoleratedTaints
  expr: count(kube_pod_info{node=~".*gpu.*"} unless on(pod) kube_pod_spec_tolerations{key="node-role.kubernetes.io/gpu"} > 0
  for: 15m
  labels:
    severity: critical
```

---
     
**Key Metrics to Monitor**:  
- `kube_node_spec_taints`  
- `kube_pod_spec_tolerations`  
- `nvidia_gpu_duty_cycle` 

**Debugging Tools**:  
```sh
# List nodes with taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
