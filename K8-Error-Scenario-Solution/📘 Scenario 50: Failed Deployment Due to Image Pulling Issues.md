# 📘 Scenario 50: Failed Deployment Due to Image Pulling Issues

**Category**: Container Registry & Authentication  
**Environment**: Kubernetes 1.22, Private Docker Registry (Harbor)  
**Impact**: Production deployment failure, 2-hour service disruption  

---

## Scenario Summary  
Deployments failed across multiple clusters due to misconfigured image pull secrets, preventing pods from pulling container images from a private Docker registry and causing widespread deployment failures.

---

## What Happened  
- **Registry authentication failure**:  
  - Registry migrated from Docker Hub to private Harbor instance  
  - Image pull secrets referenced old registry URL (`registry.old.corp:5000`)  
  - Docker config JSON contained expired credentials  
- **Observed symptoms**:  
  - Pods stuck in `ImagePullBackOff` or `ErrImagePull` states  
  - Events showed `Failed to pull image: unauthorized: authentication required`  
  - Cluster-wide impact as multiple teams used same base images  
- **Root discovery**:  
  - Secrets lacked `imagePullSecrets` reference in pod spec  
  - Registry credentials expired during maintenance window  
  - Network policies blocked access to new registry endpoint  

---

## Diagnosis Steps  

### 1. Check pod status and events:
```sh
kubectl get pods -A | grep -E "ImagePullBackOff|ErrImagePull"
kubectl describe pod <failing-pod> | grep -A10 Events
# Output: "Failed to pull image: unauthorized: authentication required"
```
