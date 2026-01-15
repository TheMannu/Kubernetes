# 📘 Scenario 50: Failed Deployment Due to Image Pulling Issues

**Category**: Container Registry & Authentication  
**Environment**: Kubernetes 1.22, Private Docker Registry (Harbor)  
**Impact**: Production deployment failure, 2-hour service disruption  

---

## Scenario Summary  
Deployments failed across multiple clusters due to misconfigured image pull secrets, preventing pods from pulling container images from a private Docker registry and causing widespread deployment failures.

---