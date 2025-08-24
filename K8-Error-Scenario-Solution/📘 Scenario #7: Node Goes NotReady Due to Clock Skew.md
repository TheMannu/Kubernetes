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

## Diagnosis Steps  

### 1. Verify node status:
```sh
kubectl get nodes -o wide | grep -i notready
```

### 2. Check time synchronization:
```sh
# On affected node:
timedatectl status
# Expected output showing NTP sync active
```

### 3. Measure clock skew:
```sh
# Compare node time with control plane
date -u && kubectl run -it --rm check-time --image=busybox --restart=Never -- date -u
```

### 4. Investigate NTP service:
```sh
systemctl status chronyd --no-pager
journalctl -u chronyd -n 50 --no-pager
```

---

## Root Cause  
**Time synchronization breakdown**:  
1. **NTP service failure**:  
   - `chronyd` crashed due to resource constraints  
   - No automatic restart configured  
2. **TLS certificate validation**:  
   - Kubernetes requires <30s clock skew for TLS validation  
   - 45s drift invalidated all API server communications  

---