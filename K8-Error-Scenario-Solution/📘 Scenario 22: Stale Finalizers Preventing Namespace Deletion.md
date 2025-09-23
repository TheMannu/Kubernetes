# 📘 Scenario 22: Stale Finalizers Preventing Namespace Deletion

**Category**: Resource Lifecycle Management  
**Environment**: Kubernetes v1.21, Self-managed with Custom Operators  
**Impact**: Namespace stuck in Terminating for 72+ hours, blocking resource cleanup  

---

## Scenario Summary  
A namespace became permanently stuck in `Terminating` state due to orphaned finalizers from an uninstalled custom controller, requiring manual intervention to resolve.

---

## What Happened  
- **Controller uninstallation**:  
  - Team deleted a CustomResourceDefinition (CRD) without first cleaning up instances  
  - Operator deployment was removed while finalizers were still pending  
- **Termination deadlock**:  
  - `kubectl delete ns` command hung indefinitely  
  - Namespace status showed `phase: Terminating` for 3 days  
  - Blocked CI/CD pipelines needing namespace reuse  
- **Root discovery**:  
  - Finalizer `custom-controller/cleanup` referenced a non-existent controller  
  - API server continuously retried finalizer execution  

---

## Diagnosis Steps  

### 1. Verify namespace status:
```sh
kubectl get ns <namespace> -o jsonpath='{.status.phase}'
# Output: Terminating
```

### 2. Inspect finalizers:
```sh
kubectl get ns <namespace> -o json | jq '.spec.finalizers'
# Showed ["kubernetes", "custom-controller/cleanup"]
```

### 3. Check for controller:
```sh
kubectl get deploy -n operator-system custom-controller
# Error: Error from server (NotFound)
```

### 4. Audit CRD history:
```sh
kubectl get crd -A --show-labels | grep custom-resource
# No output (CRD deleted)
```

---

## Root Cause  
**Lifecycle violation**:  
1. Finalizer deadlock from missing controller  
2. No pre-deletion hook to clean CR instances  
3. Namespace finalizer garbage collection blocked  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Force remove finalizers
kubectl patch ns <namespace> -p '{"spec":{"finalizers":[]}}' --type=merge

# 2. Verify deletion completes
kubectl get ns <namespace> --watch
```

### Proper Cleanup Procedure:
```sh
# For future uninstalls:
kubectl delete all --all -n <namespace>  # Delete standard resources
kubectl delete <crd-name> --all -n <namespace>  # Delete CR instances 
kubectl delete crd <crd-name>  # Remove CRD last
```

---

## Lessons Learned  
⚠️ **Finalizers are promises**: Must have operational controllers  
⚠️ **Order matters**: Always delete resources before their definitions  
⚠️ **Namespace termination is atomic**: Single finalizer blocks everything  

---

## Prevention Framework  

### 1. Uninstall Automation
```yaml
# Helm pre-delete hook
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    "helm.sh/hook": pre-delete
spec:
  template:
    spec:
      containers:
      - name: cleanup
        command: ["/bin/sh", "-c", "kubectl patch $(kubectl get <crd> -o name) -p '{\"metadata\":{\"finalizers\":[]}}' --type=merge"]
```
