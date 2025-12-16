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

### 2. Check admission controllers:
```sh
kubectl -n kube-system get pod -l component=kube-apiserver -o yaml | \
  grep -A5 "\-\-enable-admission-plugins"
# Missing "PodSecurityPolicy" in list
```

### 3. Audit PSP configuration:
```sh
kubectl get psp
# Showed restrictive-psp with allowPrivilegeEscalation: false
```

### 4. Check namespace annotations:
```sh
kubectl get namespace default -o yaml | yq '.metadata.annotations'
# Missing: pod-security.kubernetes.io/enforce
```

---

## Root Cause  
**Admission controller gap**:  
1. PSP admission controller not enabled in API server  
2. Missing namespace enforcement annotations  
3. No validation of security policy application  

---

## Fix/Workaround  

### Immediate Security Remediation:
```sh
# 1. Delete privileged pods
kubectl delete pod --all --field-selector spec.containers[].securityContext.privileged=true

# 2. Update API server configuration (EKS managed requires support ticket)
# Submit AWS support ticket to enable PodSecurityPolicy admission controller

# 3. Apply namespace-level enforcement
kubectl annotate namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest
```

### Alternative (EKS-specific):
```yaml
# EKS managed node group configuration for Pod Security Standards
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-west-2
iam:
  withOIDC: true
addons:
- name: aws-pod-identity-webhook
  version: v1.0.0
```

---

## Lessons Learned  
⚠️ **PSPs require admission controllers**: Policies alone don't enforce security  
⚠️ **Managed services have limitations**: EKS requires explicit PSP enablement  
⚠️ **Security is multi-layered**: Need runtime, admission, and namespace controls  

---

## Prevention Framework  

### 1. Admission Controller Validation
```sh
# Health check script for admission controllers
check_admission_controllers() {
  local api_pod=$(kubectl -n kube-system get pod -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')
  kubectl -n kube-system exec $api_pod -- kube-apiserver --help | grep -A100 "enable-admission-plugins"
  
  if ! kubectl -n kube-system exec $api_pod -- sh -c 'ps aux | grep kube-apiserver | grep -q PodSecurityPolicy'; then
    echo "ERROR: PodSecurityPolicy admission controller not enabled"
    exit 1
  fi
}
```
