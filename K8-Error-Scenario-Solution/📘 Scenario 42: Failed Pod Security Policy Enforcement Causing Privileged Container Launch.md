# 📘 Scenario #42: Failed Pod Security Policy Enforcement Causing Privileged Container Launch

**Category**: Cluster Security  
**Environment**: Kubernetes 1.22, AWS EKS  
**Impact**: Privileged container execution bypassing security controls  

---

## Scenario Summary  
A PodSecurityPolicy (PSP) enforcement failure allowed privileged containers to run despite restrictive policies, creating a critical security vulnerability where containers could escape isolation.

---