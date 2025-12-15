# 📘 Scenario #42: Failed Pod Security Policy Enforcement Causing Privileged Container Launch

**Category**: Cluster Security  
**Environment**: Kubernetes 1.22, AWS EKS  
**Impact**: Privileged container execution bypassing security controls  

---

## Scenario Summary  
A PodSecurityPolicy (PSP) enforcement failure allowed privileged containers to run despite restrictive policies, creating a critical security vulnerability where containers could escape isolation.

---

## What Happened  
- **Security policy bypass**:  
  - PSP defined to deny `privileged: true` and `hostPID: true`  
  - Container with `securityContext.privileged: true` deployed successfully  
- **Investigation findings**:  
  - API server admission controllers missing `PodSecurityPolicy`  
  - RBAC bindings existed but no enforcement mechanism  
  - No audit logs for PSP validation attempts  
- **Risk exposure**:  
  - Container could access host resources, devices, and processes  
  - Potential privilege escalation vectors exposed  

---

## Diagnosis Steps  

### 1. Verify pod security context:
```sh
kubectl get pod <pod-name> -o yaml | yq '.spec.containers[].securityContext.privileged'
# Output: true (should have been denied)
```
