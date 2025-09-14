# 📘 Scenario 18: kubelet Unable to Pull Images Due to Proxy Misconfig

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.25, Corporate Proxy Environment  
**Impact**: Deployment failures across 200+ nodes, 4-hour service disruption  

---

## Scenario Summary  
A missing `NO_PROXY` configuration caused kubelet to route all container image pulls through an external proxy, breaking internal registry access and cluster DNS resolution.

---

## What Happened  
- **Proxy enforcement rollout**:  
  - Corporate policy mandated HTTP_PROXY for all outbound traffic  
  - Kubelet config updated with `HTTP_PROXY=http://proxy.corp:3128`  
- **Failure symptoms**:  
  - New pods stuck in `ImagePullBackOff`  
  - `kubelet` logs showed `Failed to pull image: context deadline exceeded`  
  - Internal registry metrics showed 100% connection timeouts  
- **Network analysis**:  
  - Internal `10.0.100.0/24` traffic was being routed externally  
  - CoreDNS queries to `kubernetes.default.svc` timed out  

---

## Diagnosis Steps  

### 1. Verify image pull failures:
```sh
kubectl get pods -A -o wide | grep -E 'ImagePullBackOff|ErrImagePull'
```

### 2. Inspect kubelet proxy config:
```sh
systemctl show kubelet --property=Environment --no-pager
# Output showed HTTP_PROXY set but NO_PROXY missing
```

### 3. Test cluster DNS through proxy:
```sh
kubectl run -it --rm netcheck --image=busybox -- \
  sh -c "wget -O- http://kubernetes.default.svc --proxy=on"
# Failed with 504 Gateway Timeout
```
