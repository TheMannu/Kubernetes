# 📘 Scenario 35: ClusterConfigMap Deleted by Accident Bringing Down Addons

**Category**: Cluster Configuration  
**Environment**: Kubernetes 1.24, Rancher 2.6  
**Impact**: Critical system addons (DNS, metrics, service mesh) failed for 45 minutes  

---

## Scenario Summary  
Accidental deletion of the `kube-root-ca.crt` ConfigMap caused widespread failures across system workloads that relied on it for TLS trust, breaking core cluster functionality.

---