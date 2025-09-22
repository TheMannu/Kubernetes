# 📘 Scenario 22: Stale Finalizers Preventing Namespace Deletion

**Category**: Resource Lifecycle Management  
**Environment**: Kubernetes v1.21, Self-managed with Custom Operators  
**Impact**: Namespace stuck in Terminating for 72+ hours, blocking resource cleanup  

---

## Scenario Summary  
A namespace became permanently stuck in `Terminating` state due to orphaned finalizers from an uninstalled custom controller, requiring manual intervention to resolve.

---