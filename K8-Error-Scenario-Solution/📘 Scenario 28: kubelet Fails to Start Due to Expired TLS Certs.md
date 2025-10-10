# 📘 Scenario 28: kubelet Fails to Start Due to Expired TLS Certs

**Category**: Cluster Authentication  
**Environment**: Kubernetes 1.19, kubeadm, On-prem  
**Impact**: 40% nodes remained `NotReady` after maintenance window  

---

## Scenario Summary  
Nodes failed to rejoin the cluster after a scheduled reboot when kubelet client certificates expired during the downtime, breaking authentication with the API server.

---