# 📘 Scenario 18: kubelet Unable to Pull Images Due to Proxy Misconfig

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.25, Corporate Proxy Environment  
**Impact**: Deployment failures across 200+ nodes, 4-hour service disruption  

---

## Scenario Summary  
A missing `NO_PROXY` configuration caused kubelet to route all container image pulls through an external proxy, breaking internal registry access and cluster DNS resolution.

---