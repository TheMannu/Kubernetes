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
