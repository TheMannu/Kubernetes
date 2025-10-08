# 📘 Scenario #27: Node Bootstrap Failure Due to Unavailable Container Registry

**Category**: Cluster Provisioning  
**Environment**: Kubernetes 1.21, On-prem, Air-gapped Registry  
**Impact**: 100% node provisioning failure during registry outage  

---

## Scenario Summary  
A private container registry outage prevented new nodes from joining the cluster by blocking pulls of critical bootstrap images (`pause`, `kube-proxy`, CNI), causing complete failure of scaling operations.

---