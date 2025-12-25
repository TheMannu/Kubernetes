# 📘 Scenario 45: DNS Resolution Failure in Multi-Cluster Setup

**Category**: Multi-Cluster Networking  
**Environment**: Kubernetes 1.23, Multi-Cluster Federation with KubeFed  
**Impact**: Cross-cluster service communication failures affecting distributed applications  

---

## Scenario Summary  
DNS resolution failures between federated clusters prevented services from discovering each other, breaking distributed applications that relied on cross-cluster communication.

---

## What Happened  
- **Federation setup**:  
  - Two clusters (us-west, eu-central) federated using KubeFed  
  - Services exported via `FederatedService` resources  
- **DNS failures**:  
  - Queries to `service-name.namespace.svc.clusterset.local` timed out  
  - CoreDNS logs showed `NXDOMAIN` for federated service records  
  - Application logs showed `Name or service not known` errors  
- **Impact**:  
  - Global load balancing failed  
  - Active-active database replication broken  
  - Cross-region microservice communication disrupted  

---
