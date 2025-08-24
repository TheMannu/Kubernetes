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

## Fix/Workaround  

### Immediate Recovery:
```sh
# Force time sync (node)
sudo chronyc makestep
sudo systemctl restart chronyd

# Restart kubelet after sync
sudo systemctl restart kubelet
```

### Long-term Solutions:
1. **Configure NTP hardening**:
   ```ini
   # /etc/chrony.conf
   pool pool.ntp.org iburst
   makestep 1.0 3
   rtcsync
   ```

2. **Add startup probe**:
   ```yaml
   # kubelet systemd unit override
   [Unit]
   After=chronyd.service
   Requires=chronyd.service
   ```

---

## Lessons Learned  
⚠️ **TLS is time-sensitive**: Kubernetes components require sub-30s synchronization  
⚠️ **Silent failure mode**: NTP services can fail without obvious symptoms until TLS breaks  

---

## Prevention Framework  

### 1. Time Synchronization
```sh
# Deploy NTP as DaemonSet (for air-gapped environments)
kubectl apply -f https://k8s.io/examples/admin/chrony-daemonset.yaml
```

### 2. Monitoring
```yaml
# Prometheus alert for clock skew
- alert: NodeClockSkew
  expr: abs(time() - node_time_seconds{job="node-exporter"}) > 30
  for: 5m
  labels:
    severity: critical
```

### 3. Node Configuration
```ini
# /etc/systemd/timesyncd.conf (alternative)
[Time]
NTP=pool.ntp.org
FallbackNTP=time.google.com
```

### 4. Validation Checks
```sh
# Pre-flight check for new nodes
kubeadm join ... --ignore-preflight-errors=all  # NEVER do this
# Instead fix NTP before joining
```

---

**Key Metrics to Monitor**:  
- `node_timex_offset_seconds`  
- `node_timex_sync_status`  
- `kubelet_node_name{condition="Ready"}`  

**Tools for Investigation**:  
```sh
chronyc tracking     # Check NTP sync status
chronyc sources -v   # Verify NTP sources
openssl s_client -connect api-server:6443 # TLS handshake test
```

