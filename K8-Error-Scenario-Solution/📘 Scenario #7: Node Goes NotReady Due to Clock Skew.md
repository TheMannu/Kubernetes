# 📘 Scenario #7: Node Goes NotReady Due to Clock Skew

**Category**: Cluster Reliability  
**Environment**: Kubernetes v1.20, On-prem (VMware)  
**Impact**: Node isolation, workload rescheduling  

---

## Scenario Summary  
A worker node became `NotReady` due to TLS certificate validation failures caused by significant clock drift (45 seconds), resulting from a failed NTP service.

---

## What Happened  
- **Gradual node degradation**:  
  - Initial intermittent `NodeReady` condition flapping  
  - Progressed to permanent `NotReady` state  
- **TLS handshake failures**:  
  ```log
  x509: certificate has expired or is not yet valid: current time 2023-05-02T15:45:30Z is after 2023-05-02T15:44:45Z
  ```  
- **Workload impact**:  
  - Pods marked for eviction after 5-minute `NotReady` threshold  
  - Cluster autoscaler provisioned replacement nodes  

---