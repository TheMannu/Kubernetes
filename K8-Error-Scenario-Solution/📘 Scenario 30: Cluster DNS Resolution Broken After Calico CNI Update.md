# 📘 Scenario 30: Cluster DNS Resolution Broken After Calico CNI Update

**Category**: Cluster Networking  
**Environment**: Kubernetes 1.23, Self-managed Calico 3.22  
**Impact**: Cluster-wide DNS outage lasting 53 minutes  

---

## Scenario Summary  
A Calico CNI upgrade introduced default-deny network policies that inadvertently blocked CoreDNS egress traffic, breaking all DNS resolution across the cluster.

---