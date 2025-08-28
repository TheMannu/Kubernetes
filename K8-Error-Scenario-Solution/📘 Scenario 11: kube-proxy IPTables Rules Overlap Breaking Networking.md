# 📘 Scenario #11: kube-proxy IPTables Rules Overlap Breaking Networking

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.22, On-prem, kube-proxy in IPTables mode  
**Impact**: Service connectivity loss, DNS resolution failures  

---

## Scenario Summary  
Custom IPTables NAT rules installed on nodes conflicted with kube-proxy's service routing rules, causing broken cluster networking and service blackholes.  

---

## What Happened  
- **Custom routing requirement**:  
  - Admin added NAT rules for external traffic redirection  
  - Rules modified `KUBE-SERVICES` chain directly  
- **Network symptoms**:  
  - Intermittent `Connection refused` for ClusterIP services  
  - CoreDNS resolution failures (`SERVFAIL`)  
  - NodePort services became unreachable  
- **kube-proxy behavior**:  
  - Continuous `Failed to ensure iptables rules` log warnings  
  - Rule ordering became corrupted after kube-proxy syncs  

---
