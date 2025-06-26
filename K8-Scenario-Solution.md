# 📘 Scenario 1: Zombie Pods Causing Node Drain to Hang

**Category**: Cluster Management  
**Environment**: Kubernetes v1.23, On-prem bare metal, Systemd cgroups 

## Scenario Summary  
Node drain operation stuck indefinitely due to an unresponsive terminating pod with a custom finalizer.

## What Happened  
- A pod with a custom finalizer failed to complete termination  
- `kubectl drain` command hung indefinitely  
- The pod remained in "Terminating" state even after being marked for deletion  
- API server waited indefinitely as the finalizer wasn't removed  

## Diagnosis Steps  
1. Checked for lingering pods:  
   ```sh
   kubectl get pods --all-namespaces -o wide
   ```
2. Identified pod stuck in `Terminating` state for >20 minutes
3. Inspected pod details to find custom finalizer:  
   ```sh
   kubectl describe pod <pod>
   ```
4. Discovered the controller managing the finalizer had crashed  

## Root Cause  
Finalizer logic failed to execute because its controller was down, making the pod undeletable.

## Fix/Workaround  
Force-remove the finalizer:  
```sh
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
```

## Lessons Learned  
- Finalizers should implement timeout/fail-safe mechanisms  
- Critical to monitor controller health when using finalizers 

## How to Avoid  
✅ Avoid finalizers unless absolutely necessary  
✅ Implement monitoring for stuck `Terminating` pods  
✅ Add retry/timeout logic in finalizer controllers  
✅ Consider pod disruption budgets for critical workloads


Key improvements:
1. Better structure with clear section headers
2. Added code formatting for commands
3. Improved readability with bullet points
4. Added checkmark emojis for prevention measures
5. Consistent spacing and formatting
6. Added a missing prevention measure (pod disruption budgets)

---

# 📘 Scenario 2: API Server Crash Due to Excessive CRD Writes  

**Category**: Cluster Management  
**Environment**: Kubernetes v1.24, GKE, Heavy use of custom controllers  

---

## Scenario Summary  
API server crashed after being flooded by a malfunctioning controller creating excessive Custom Resources (CRs).  

---

## What Happened  
- A buggy controller entered an infinite reconciliation loop, creating **thousands of duplicate CRs**  
- **etcd overwhelmed** with write requests, causing high latency  
- API server became unresponsive, returning `504 Gateway Timeout` errors  
- Cluster operations (including `kubectl` commands) failed due to backend pressure  

---

## Diagnosis Steps  
1. **Observed symptoms**:  
   - API latency spikes  
   - `kubectl` commands timing out with `504` errors  
2. **Checked CRD volume**:  
   ```sh
   kubectl get crds | wc -l  # Excessive count confirmed
   ```  
3. **Traced controller logs**:  
   - Found **infinite reconcile loop** on a specific CR type  
4. **Monitored etcd**:  
   - Disk I/O saturated at 100%  
   - **High memory usage** from storing redundant CRs  

---

## Root Cause  
**Faulty reconciliation logic**:  
- Controller **always executed `create`** instead of checking existing resources  
- No rate-limiting or deduplication safeguards  

---

## Fix/Workaround  
1. **Immediate mitigation**:  
   ```sh
   kubectl scale deploy <controller> --replicas=0  # Stop the controller
   ```  
2. **Cleanup CRs**:  
   - Used batch deletion (e.g., `kubectl delete crd --all` or selector-based cleanup)  

---

## Lessons Learned  
⚠️ **Always test reconciliation logic** in a non-production cluster first.  
⚠️ **Monitor etcd metrics** (I/O, memory) to catch flooding early.  

---

## How to Avoid  
✅ **Add guards in reconciliation logic**:  
   - Check `resourceVersion` before creation  
   - Implement idempotent operations  
✅ **Set Prometheus alerts for**:  
   - CR count per namespace  
   - etcd write latency  
✅ **Use admission webhooks** to enforce CR quotas  
✅ **Enable etcd compaction** to reduce storage bloat    

---

### Key Improvements:  
1. **Structured sections** with clear separation (`---`)  
2. **Code blocks** for commands and critical diagnostics  
3. **Emoji/icons** (⚠️, ✅) for visual scannability
4. **Added prevention tips**:  
   - Admission webhooks  
   - etcd compaction  
5. **Concise root cause** highlighting the logic flaw  
6. **Formatted fixes** as actionable steps 

---

# 📘 Scenario 3: Node Not Rejoining After Reboot

**Category**: Cluster Management  
**Environment**: Kubernetes v1.21, Self-managed cluster, Static nodes  

---

## Scenario Summary  
A rebooted node failed to rejoin the cluster due to a kubelet identity mismatch caused by hostname changes.

---

## What Happened  
- After a **kernel upgrade and reboot**, a node disappeared from `kubectl get nodes`  
- **Kubelet failed to register** with the control plane  
- Investigation revealed a **hostname mismatch** between the node's current state and its cluster registration  

---

## Diagnosis Steps  
1. **Checked system logs**:  
   ```sh
   journalctl -u kubelet --no-pager | grep -i "registration"
   ```
2. **Verified kubelet configuration**:  
   - Found `--hostname-override` didn't match the originally registered node name 
3. **Compared cluster state**:  
   ```sh
   kubectl get nodes -o wide  # Showed old hostname vs. current DHCP-assigned hostname
   ```
4. **Identified root issue**:  
   - Node's **hostname changed after reboot** (DHCP lease renewal or cloud-init behavior)  

---

## Root Cause  
**Identity mismatch**:  
- Kubelet attempted to register with a **new hostname**  
- Control plane still expected the **original node identity**  
- Certificate SANs/Node authorization rejected the "new" node  

---

## Fix/Workaround  
1. **Corrected hostname override**:  
   ```sh
   # Update kubelet config (e.g., /etc/default/kubelet)
   KUBELET_EXTRA_ARGS="--hostname-override=<original-node-name>"
   systemctl restart kubelet
   ```
2. **Cleaned stale node entry** (if needed):  
   ```sh
   kubectl delete node <old-node-name>
   ```

---

## Lessons Learned  
⚠️ **Hostname consistency is critical** for node identity in Kubernetes.  
⚠️ **DHCP + Kubernetes requires careful planning** to avoid identity drift.  

---

## How to Avoid  
✅ **Use static hostnames/IPs** for nodes:  
   ```sh
   hostnamectl set-hostname <persistent-name>
   ```
✅ **Standardize node provisioning** with:  
   - Immutable cloud-init configurations  
   - Kubeadm `--node-name` flag (if applicable) 
✅ **Monitor node heartbeats**:  
   ```sh
   kubectl get nodes -w
   ```
✅ **Implement automated detection** for:  
   - Nodes stuck in `NotReady`  
   - Hostname mismatches in kubelet logs  


---

### Key Improvements:
1. **Added actionable commands** with proper syntax highlighting
2. **Structured troubleshooting flow** from symptoms to resolution
3. **Included prevention automation** (monitoring commands)
4. **Emphasized key takeaways** with icons (⚠️, ✅)
5. **Added context** about certificate SANs/authorization impact
6. **Standardized formatting** with clear section breaks (`---`)

---
# 📘 Scenario 4: Etcd Disk Full Causing API Server Timeout

**Category**: Cluster Management  
**Environment**: Kubernetes v1.25, Bare-metal cluster  

---

## Scenario Summary  
Cluster API server became unresponsive due to etcd running out of disk space from accumulated WAL logs and uncompacted revisions.

---

## What Happened  
- Cluster began **failing API requests** with timeout errors  
- **Etcd logs** showed `"mvcc: database space exceeded"` errors  
- **API server logs** indicated failed storage operations (`"etcdserver: request timed out"`)  
- Cluster operations (including `kubectl` commands) became unreliable  

---

## Diagnosis Steps  
1. **Verified disk space** on etcd nodes:  
   ```sh
   df -h /var/lib/etcd
   ```
2. **Analyzed etcd storage usage**:  
   ```sh
   sudo du -sh /var/lib/etcd/member/
   ```
3. **Checked etcd metrics**:  
   ```sh
   etcdctl endpoint status --write-out=table
   ```
4. **Identified root issues**:  
   - **No compaction** of historical revisions  
   - **Accumulated WAL logs** and snapshots  
   - **No disk space alerts** configured  

---

## Root Cause  
**Storage exhaustion due to**:  
1. Lack of **automatic compaction** of old revisions  
2. No **routine defragmentation** of etcd database  
3. Insufficient **disk space monitoring**  

---

## Fix/Workaround  

### Immediate Recovery:
1. **Compact old revisions** (replace `<rev>` with current revision):  
   ```sh
   etcdctl compact $(etcdctl endpoint status --write-out=json | jq -r '.[].Status.header.revision')
   ```
2. **Defragment etcd database**:  
   ```sh
   etcdctl defrag --cluster
   ```
3. **Clean up disk space**:  
   ```sh
   # Remove old snapshots/WAL logs (if safe)
   sudo find /var/lib/etcd/member/wal -type f -mtime +7 -delete
   ```

### Long-term Solution:
- **Increase etcd volume size**  
- **Configure automatic maintenance** (see prevention below)  

---

## Lessons Learned  
⚠️ **Etcd is stateful**: Requires proactive disk management.  
⚠️ **Silent failures**: API server errors may not clearly indicate etcd issues.  

---

## How to Avoid  
✅ **Enable automatic compaction** (e.g., in etcd config):  
   ```yaml
   auto-compaction-mode: periodic
   auto-compaction-retention: "1h"  # Adjust based on cluster size
   ```
✅ **Schedule regular defragmentation** (cronjob or script):  
   ```sh
   etcdctl defrag --cluster
   ```
✅ **Monitor critical metrics**:  
   - etcd storage size (`etcd_mvcc_db_total_size_in_bytes`)  
   - Disk available space  
   - `etcd_server_quota_backend_bytes` (quota alerts)  
✅ **Set up alerts** for:  
   - Disk >70% utilization  
   - etcd leader changes  
   - High latency on etcd operations  

---

### Key Improvements:
1. **Added detailed recovery commands** with proper examples
2. **Structured into immediate vs long-term solutions**
3. **Included specific metrics** for monitoring
4. **Added YAML examples** for auto-compaction config
5. **Improved readability** with clear sections and code blocks
6. **Added context** about API server error propagation

---
---
# 📘 Scenario 5: Misconfigured Taints Blocking Pod Scheduling

**Category**: Cluster Management  
**Environment**: Kubernetes v1.26, Multi-tenant cluster  

---

## Scenario Summary  
Critical workloads failed to schedule after blanket `NoSchedule` taints were applied without corresponding workload tolerations, causing cluster-wide scheduling issues.

---

## What Happened  
- A team added `NoSchedule` taints to **all worker nodes** to isolate their application  
- **Existing workloads** lacked matching tolerations  
- Cluster monitoring showed:  
  - Sudden spike in `Pending` pods  
  - Critical system pods (CNI, monitoring) failing to schedule  
- Applications began failing SLA as pods couldn't be rescheduled  

---

## Diagnosis Steps  

### 1. Identified scheduling failures:
```sh
kubectl get pods --all-namespaces --field-selector status.phase=Pending
```

### 2. Analyzed pending pods:
```sh
kubectl describe pod <pending-pod> | grep -A10 Events
# Output showed: "0/3 nodes are available: 3 node(s) had untolerated taint..."
```

### 3. Inspected node taints:
```sh
kubectl get nodes -o json | jq '.items[].spec.taints'
# OR
kubectl describe node <node> | grep Taints
```

### 4. Verified RBAC:
```sh
kubectl who-can update nodes
```

---

## Root Cause  
**Improper taint management**:  
- Blanket `NoSchedule` taints applied without:  
  - **Cluster-wide impact assessment**  
  - **Required tolerations** in existing workloads  
- **No change control process** for node modifications  

---

## Fix/Workaround  

### Immediate Action:
```sh
# Remove problematic taints from all nodes
kubectl taint nodes --all <taint-key>-
```

### Verify Recovery:
```sh
watch kubectl get pods -A -o wide  # Observe pods transitioning to Running
```

### Long-term Solution:
1. Implement **selective tainting** (only specific nodes)  
2. Add **required tolerations** to system-critical deployments  

---

## Lessons Learned  
⚠️ **Taints are cluster-wide weapons**: Affects all workloads without tolerations  
⚠️ **System workloads need special consideration**: CNI, CSI, monitoring pods must tolerate common taints  

---

## How to Avoid  

✅ **Education & Documentation**:  
   - Conduct workshops on taints/tolerations  
   - Maintain a cluster taint registry  

✅ **Technical Safeguards**:  
   ```sh
   # Use kubectl diff to preview taint changes
   kubectl taint nodes --dry-run=server ...
   ```
   
✅ **RBAC Controls**:  
   ```yaml
   # Restrict node modification to cluster-admins
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: restricted-node-access
   rules:
   - apiGroups: [""]
     resources: ["nodes"]
     verbs: ["get", "list", "watch"]  # No update/patch
   ```

✅ **Admission Controls**:  
   - Use OPA/Gatekeeper to:  
     - Require matching tolerations for certain taints  
     - Prevent blanket taints on all nodes  

✅ **Monitoring**:  
   - Alert on:  
     - High `Pending` pod count  
     - Node taint changes  
     - System pods not running  

### Key Improvements:
1. **Added concrete commands** for diagnosis and remediation
2. **Structured RBAC examples** for prevention
3. **Included admission control** strategies
4. **Added monitoring recommendations**
5. **Separated immediate vs long-term solutions**
6. **Emphasized system workload requirements**

---
---

# 📘 Scenario 6: Kubelet DiskPressure Loop on Large Image Pulls

**Category**: Node Resource Management  
**Environment**: Kubernetes v1.22, EKS  

## Scenario Summary  
Nodes entered a destructive eviction loop due to disk exhaustion from pulling oversized container images, triggering continuous `DiskPressure` conditions.

---

## What Happened  
- A **multi-layered container image (5GB+)** was deployed cluster-wide  
- Nodes exhibited:  
  - Frequent pod evictions (`Evicted: DiskPressure`)  
  - Cascading failures as evicted pods automatically rescheduled  
  - **CPU throttling** from kubelet's constant garbage collection  
- Cluster autoscaler added nodes but they quickly became unstable  

---

## Diagnosis Steps  

### 1. Verified node conditions:
```sh
kubectl get nodes -o json | jq '.items[].status.conditions'
# Output showed DiskPressure=True
```

### 2. Analyzed disk usage:
```sh
kubectl debug node/<node> -it --image=busybox -- df -h /var/lib/containerd
```

### 3. Inspected image cache:
```sh
kubectl debug node/<node> -it --image=ubuntu -- crictl images
# Showed multiple large images with duplicate layers
```

### 4. Checked kubelet logs:
```sh
journalctl -u kubelet --no-pager | grep -i "DiskPressure"
# Revealed continuous image garbage collection attempts
```

---

## Root Cause  
**Image storage bloat from**:  
1. **Unoptimized image** with redundant layers (dev tools left in production image)  
2. **No image size limits** in CI/CD pipeline  
3. **Insufficient disk headroom** for normal operations  

---

## Fix/Workaround  

### Immediate Action:
```sh
# Manually clean up node images (on affected nodes)
crictl rmi --prune
```

### Long-term Solution:
1. **Optimized Dockerfile**:
   ```dockerfile
   # Before (1.2GB image)
   FROM ubuntu:20.04
   RUN apt update && apt install -y build-essential... 
   
   # After (280MB image using multi-stage)
   FROM ubuntu:20.04 as builder
   RUN apt update && apt install -y build-essential...
   
   FROM gcr.io/distroless/base
   COPY --from=builder /app /app
   ```

2. **Node Configuration**:
   ```yaml
   # kubelet arguments (increase thresholds)
   --eviction-hard=memory.available<500Mi,nodefs.available<15%
   --image-gc-high-threshold=85
   --image-gc-low-threshold=80
   ```

---

## Lessons Learned  
⚠️ **DiskPressure is cascading**: One bloated image can destabilize entire nodes  
⚠️ **Eviction loops compound problems**: Rescheduled pods often re-pull the same images  

---

## How to Avoid  

✅ **Image Optimization**:  
   - Enforce multi-stage builds  
   - Use `dive` to analyze layer efficiency  
   ```sh
   dive build -t <image> .
   ```

✅ **Admission Controls**:  
   ```yaml
   # OPA/Gatekeeper policy example
   apiVersion: constraints.gatekeeper.sh/v1beta1
   kind: K8sAllowedRepos
   metadata:
     name: prod-image-size-limit
   spec:
     match:
       kinds:
       - apiGroups: [""]
         kinds: ["Pod"]
     parameters:
       maxSizeMB: 500
   ```

✅ **Node Configuration**:  
   - Separate container runtime partition (>100GB recommended)  
   - Pre-pull critical images during node bootstrap  

✅ **Monitoring**:  
   - Alert on:  
     - `kubelet_volume_stats_available_bytes` < 20%  
     - `container_fs_usage_bytes` approaching capacity  
     - Pod eviction rate spikes  

✅ **CI/CD Safeguards**:  
   ```sh
   # Fail pipeline if image exceeds size limit
   docker inspect --format='{{.Size}}' $IMAGE | awk '{if ($1 > 500000000) exit 1}'
   ```

---

### Key Improvements:
1. **Added concrete Dockerfile examples** showing optimization
2. **Included kubelet configuration parameters** for disk management
3. **Provided OPA/Gatekeeper policy** for enforcement
4. **Added CI/CD pipeline check** example
5. **Expanded monitoring metrics**
6. **Separated immediate vs long-term solutions**

---

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

# 📘 Scenario #8: API Server High Latency Due to Event Flooding

**Category**: API Server Performance  
**Environment**: Kubernetes v1.23, Azure AKS  
**Impact**: Cluster-wide API latency spikes, scheduler disruptions  

---

## Scenario Summary  
A malfunctioning controller flooded the cluster with 50+ events/second, overwhelming etcd and causing API server latency to exceed 5s.

---

## What Happened  
- **API degradation**:  
  - `kubectl` commands timed out (`Error from server (Timeout)`)  
  - Scheduler could not update pod status (`leader election lost`)  
- **Event storm**:  
  - 500,000+ events in default namespace  
  - etcd WAL growth rate spiked to 50MB/minute  
- **Controller misbehavior**:  
  - Continuous `FailedCreate` events for non-critical conditions  

---

## Diagnosis Steps  

### 1. Identify event source:
```sh
kubectl get events --all-namespaces --sort-by='.metadata.creationTimestamp' | \
  awk '{print $2}' | sort | uniq -c | sort -n
```

### 2. Check etcd metrics:
```sh
# Via Prometheus
etcd_disk_wal_fsync_duration_seconds{quantile="0.99"} > 1
etcd_server_quota_backend_bytes{cluster="true"} > 90%
```

### 3. Locate problematic controller:
```sh
kubectl logs -n <namespace> <controller-pod> | \
  grep -i "event.*failed" -m 20
```

### 4. Verify API server health:
```sh
kubectl get --raw='/metrics' | \
  grep -E 'apiserver_request_duration_seconds|apiserver_current_inflight_requests'
```

---

## Root Cause  
**Uncontrolled event generation**:  
1. **No rate limiting**: Controller logged identical errors in tight loop  
2. **No deduplication**: Same event recorded hundreds of times  
3. **No cleanup**: Default event TTL of 1h insufficient  

---

## Fix/Workaround  

### Immediate Actions:
```sh
# Bulk delete events (carefully!)
kubectl delete events --all --namespace=default

# Scale down offending controller
kubectl scale deploy <controller> --replicas=0
```

### Controller Patch Example:
```go
// Before: Uncontrolled events
recorder.Eventf(obj, "Warning", "FailedCreate", "Error creating resource")

// After: Rate-limited with deduplication
if time.Since(lastEvent) > 5*time.Minute {
    recorder.Eventf(obj, "Warning", "FailedCreate", 
        "Error creating resource (repeated %d times)", errorCount)
    lastEvent = time.Now()
    errorCount = 0
}
```

---

## Lessons Learned  
⚠️ **Events are etcd writes**: Each event consumes etcd I/O capacity  
⚠️ **Cascading failures**: API latency affects all controllers/schedulers  

---

## Prevention Framework  

### 1. Cluster Configuration
```yaml
# API server flags
--event-ttl=30m  # Reduce from default 1h
--enable-admission-plugins=EventRateLimit
--admission-control-config-file=eventconfig.yaml
```

### 2. Event Rate Limiting Policy
```yaml
# eventconfig.yaml
apiVersion: eventratelimit.admission.k8s.io/v1alpha1
kind: Configuration
limits:
  - type: Namespace
    qps: 50
    burst: 100
```

### 3. Controller Best Practices
```go
// Use exponential backoff for error events
eventInterval := math.Min(5*math.Pow(2, retryCount), 300)
if time.Since(lastEvent) > time.Duration(eventInterval)*time.Second {
    recordEvent()
}
```

### 4. Monitoring
```yaml
# Critical Prometheus alerts
- alert: EventStorm
  expr: rate(apiserver_event_count[1m]) > 100
  for: 2m
  labels:
    severity: critical
```

---

**Key Metrics to Watch**:  
- `apiserver_storage_objects{resource="events"}`  
- `etcd_mvcc_db_total_size_in_bytes`  
- `apiserver_request_duration_seconds{verb="POST",resource="events"}`  

**Debugging Tools**:  
```sh
# Real-time event monitoring
kubectl get events -w --field-selector type=Warning

# etcd compaction (if needed)
etcdctl compact $(etcdctl endpoint status -w json | jq -r '.[].Status.header.revision')
```
---

# 📘 Scenario #9: CoreDNS CrashLoop on Startup  

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.24, DigitalOcean  
**Impact**: Cluster-wide DNS resolution failure  

---

## Scenario Summary  
CoreDNS pods entered a `CrashLoopBackOff` state due to an invalid `Corefile` configuration, breaking DNS resolution across all cluster workloads.  

---

## What Happened  
- **Custom `Corefile` update**:  
  - A new `rewrite` rule was added with incorrect syntax  
  - Applied via `kubectl edit configmap/coredns -n kube-system`  
- **Immediate failure**:  
  - CoreDNS pods crashed on startup (`Error in Corefile: invalid directive`)  
  - Cluster services began failing with `NXDOMAIN` errors  
- **Cascading effects**:  
  - `kubectl` commands slowed due to DNS timeouts  
  - Service mesh (Istio) sidecars failed health checks  

---

## Diagnosis Steps  

### 1. Check CoreDNS pod status:
```sh
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Output showed CrashLoopBackOff
```

### 2. Inspect pod logs:
```sh
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# Error: "Corefile:5 - Error during parsing: Unknown directive 'rewrit'"
```

### 3. Verify ConfigMap:
```sh
kubectl get configmap/coredns -n kube-system -o yaml | grep -A10 "Corefile"
# Revealed malformed rewrite rule:
# rewrit name exact foo.bar internal.svc.cluster.local
```

### 4. Validate Corefile syntax (offline):
```sh
docker run -i coredns/coredns:1.8.6 -conf - <<< "$(kubectl get configmap/coredns -n kube-system -o jsonpath='{.data.Corefile}')"
```

---

## Root Cause  
**Configuration error**:  
- Typo in directive (`rewrit` vs `rewrite`)  
- Missing required plugin (`rewrite` not in CoreDNS image)  
- No validation before applying changes  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# Revert to previous ConfigMap version
kubectl rollout undo configmap/coredns -n kube-system

# Force CoreDNS rollout
kubectl rollout restart deployment/coredns -n kube-system
```

### Permanent Solution:
1. **Add validation workflow**:
   ```sh
   # Pre-apply check using coredns/corerc
   docker run -i coredns/corerc:latest validate < corefile-new.cfg
   ```

2. **Implement GitOps**:
   ```yaml
   # ArgoCD Application with kustomize
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   spec:
     syncPolicy:
       automated:
         selfHeal: true  # Auto-revert broken changes
   ```

---

## Lessons Learned  
⚠️ **DNS is critical infrastructure**: Even syntax errors cause cluster-wide outages  
⚠️ **ConfigMaps need version control**: `kubectl edit` is dangerous without backups  

---

## Prevention Framework  

### 1. Validation Safeguards
```sh
# Pre-commit hook example (.git/hooks/pre-commit)
#!/bin/sh
docker run -i coredns/coredns:${CORE_DNS_VERSION} -conf - < corefile.cfg || {
  echo "Corefile validation failed"
  exit 1
}
```

### 2. Change Management
```yaml
# CoreDNS ConfigMap backup policy (Velero example)
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: coredns-daily
spec:
  schedule: "@daily"
  template:
    includedNamespaces: [kube-system]
    includedResources: [configmaps]
    labelSelector:
      matchLabels:
        k8s-app: kube-dns
```

### 3. Monitoring
```yaml
# Prometheus alerts for CoreDNS health
- alert: CoreDNSDown
  expr: absent(up{job="kube-dns"} == 1)
  for: 1m
  labels:
    severity: critical
```

### 4. Deployment Best Practices
```yaml
# Helm values.yaml safety checks
coredns:
  customCorefile: |
    import ./validate-corefile.sh  # Pre-install validation
  rollingUpdate:
    maxUnavailable: 0  # Ensure zero-downtime updates
```

---

**Key Metrics to Monitor**:  
- `coredns_cache_hits_total`  
- `coredns_panics_total`  
- `kube_dns_errors_total`  

**Debugging Tools**:  
```sh
# Live CoreDNS query test
kubectl run -it --rm dns-test --image=busybox --restart=Never -- nslookup kubernetes.default
``` 

**Backup Corefile Template**:  
```corefile
. {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload  # Enable automatic config reload
}
```

---
---

# 📘 Scenario #10: Control Plane Unavailable After Flannel Misconfiguration  

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.18, On-prem (Bare Metal), Flannel CNI  
**Impact**: Complete control plane isolation, workload communication breakdown  

---

## Scenario Summary  
A new node joined with an incorrect pod CIDR, causing Flannel routing tables to corrupt and severing all control plane communications.  

---

## What Happened  
- **Node onboarding error**:  
  - New node configured with `10.244.1.0/24` when cluster expected `10.244.0.0/16`  
  - Flannel's `kube-subnet-mgr` failed to reconcile the mismatch  
- **Network symptoms**:  
  - API server became unreachable from worker nodes (`kubectl` timeout)  
  - Cross-node pod communication failed (`No route to host`)  
  - Flannel pods logged `"Failed to acquire lease"` errors  
- **Cascading failures**:  
  - Kube-proxy iptables rules became inconsistent  
  - Node status updates stopped flowing to control plane  

---

## Diagnosis Steps  

### 1. Verify cluster networking state:
```sh
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD_CIDR:.spec.podCIDR
# Showed mismatched CIDRs across nodes

### 2. Check Flannel configuration:
```sh
kubectl -n kube-system get cm kube-flannel-cfg -o jsonpath='{.data.net-conf\.json}' | jq
# Revealed Network: "10.244.0.0/16" vs node's 10.244.1.0/24
```

### 3. Inspect network routes:
```sh
# On affected node:
ip route show | grep flannel
# Missing expected overlay routes
```

### 4. Analyze kubelet logs:
```sh
journalctl -u kubelet --no-pager | grep -i podcidr
# Log showed "PodCIDR not set for node" warnings
```

---

## Root Cause  
**CIDR misalignment**:  
1. **Manual node addition** bypassing cluster provisioning standards  
2. **No validation** of podCIDR consistency  
3. **Flannel's strict subnet requirements** not enforced  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# Drain and remove misconfigured node
kubectl drain <node> --delete-emptydir-data --force
kubectl delete node <node>

# Reset Flannel (on all nodes)
sudo systemctl restart flanneld
sudo iptables -F && sudo iptables -t nat -F

# Rejoin node with correct CIDR
kubeadm join ... --pod-network-cidr=10.244.0.0/16
```

### Long-term Solution:
1. **Automated node validation**:
   ```yaml
   # OPA Gatekeeper constraint
   apiVersion: constraints.gatekeeper.sh/v1beta1
   kind: K8sRequiredPodCIDR
   spec:
     match:
       kinds: [{kind: Node}]
     parameters:
       cidr: "10.244.0.0/16"
   ```

2. **Flannel configuration hardening**:
   ```json
   {
     "Network": "10.244.0.0/16",
     "Backend": {
       "Type": "vxlan",
       "DirectRouting": true
     }
   }
   ```

---

## Lessons Learned  
⚠️ **CNI plugins are fragile**: Flannel requires perfect CIDR alignment  
⚠️ **Manual node additions are dangerous**: Always use automation  
⚠️ **Control plane depends on pod network**: Broken CNI → Broken API  

---

## Prevention Framework  

### 1. Admission Control
```yaml
# Kyverno ClusterPolicy example
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-pod-cidr
spec:
  validationFailureAction: enforce
  rules:
  - name: check-pod-cidr
    match:
      resources:
        kinds:
        - Node
    validate:
      message: "Pod CIDR must match cluster range"
      pattern:
        spec:
          podCIDR: "10.244.0.0/16"
```

### 2. Provisioning Safeguards
```sh
# kubeadm pre-flight check
kubeadm join --config=kubeadm-config.yaml --dry-run
```

### 3. Monitoring
```yaml
# Critical Prometheus alerts
- alert: FlannelCIDRMismatch
  expr: count(kube_node_info{pod_cidr!="10.244.0.0/16"}) > 0
  labels:
    severity: critical
```

### 4. Automation Templates
```terraform
# Terraform node template
resource "local_file" "kubeadm_config" {
  content = templatefile("${path.module}/templates/kubeadm-config.tpl", {
    pod_cidr = "10.244.${count.index}.0/24"
  })
}
```

---

**Key Metrics to Monitor**:  
- `flannel_subnet_leases`  
- `kube_node_status_condition{condition="NetworkUnavailable"}`  
- `node_network_up`  

**Debugging Tools**:  
```sh
# Network connectivity test
kubectl run net-test --image=nicolaka/netshoot --rm -it -- \
   curl -m 5 https://kubernetes.default.svc.cluster.local
``` 

**Flannel Health Checks**:  
```sh
# Verify Flannel interface
ip -d link show flannel.1
# Check route propagation
etcdctl get /coreos.com/network/subnets --prefix
```

---

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

## Diagnosis Steps  

### 1. Verify service connectivity:
```sh
kubectl run net-test --image=nicolaka/netshoot --rm -it -- \
   curl -v http://kubernetes.default.svc.cluster.local
```

### 2. Inspect IPTables rules:
```sh
iptables-save -t nat | grep -A 10 KUBE-SERVICES
# Found custom SNAT rules inserted above kube-proxy rules
```

### 3. Check kube-proxy logs:
```sh
journalctl -u kube-proxy --no-pager | grep -i conflict
# "Couldn't clean up iptables rules: exit status 2"
```

### 4. Compare rule versions:
```sh
# Before custom rules
iptables-save > /tmp/pre-change.iptables
# After incident
diff /tmp/pre-change.iptables <(iptables-save)
```



## Root Cause  
**Rule precedence conflict**:  
1. Custom rules inserted at **top** of `KUBE-SERVICES` chain  
2. kube-proxy rules **overwritten** during sync cycles (every 30s)  
3. No `-I` vs `-A` distinction in rule management  

---
## Fix/Workaround  

### Emergency Recovery:
```sh
# Flush ALL custom NAT rules (CAUTION: Verify first!)
iptables -t nat -F

# Restart kube-proxy to rebuild rules
systemctl restart kube-proxy

# Verify service restoration
kubectl get svc -A -o wide
```

### Long-term Solution:
1. **Isolate custom rules**:
   ```sh
   # Create dedicated chain
   iptables -t nat -N CUSTOM-NAT
   iptables -t nat -A PREROUTING -j CUSTOM-NAT
   ```

### Long-term Solution:
1. **Isolate custom rules**:
   ```sh
   # Create dedicated chain
   iptables -t nat -N CUSTOM-NAT
   iptables -t nat -A PREROUTING -j CUSTOM-NAT
   ```

2. **kube-proxy configuration**:
   ```yaml
   # /var/lib/kube-proxy/configuration.conf
   iptables:
     minSyncPeriod: 5s  # Faster recovery from conflicts
     localhostNodePorts: false  # Reduce rule complexity
   ```

---

## Lessons Learned  
⚠️ **kube-proxy owns NAT chains**: Never manually modify `KUBE-*` chains  
⚠️ **Rule ordering matters**: First-match wins in IPTables  
⚠️ **Silent failures**: kube-proxy retries indefinitely without alerting  

---

## Prevention Framework  

### 1. Firewall Policy
```sh
# Safe rule addition template
iptables -t nat -N MY-APP-NAT
iptables -t nat -A PREROUTING -j MY-APP-NAT  # Runs BEFORE KUBE-SERVICES
iptables -t nat -A POSTROUTING -j MY-APP-NAT
```

### 2. Admission Control
```yaml
# OPA Gatekeeper policy to block node modifications
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedSysctls
metadata:
  name: deny-iptables-mod
spec:
  match:
    kinds:
    - Node
  parameters:
    forbiddenPatterns: ["*iptables*"]
```

### 3. Monitoring
```yaml
# Prometheus alerts for kube-proxy health
- alert: KubeProxyRuleSyncFailed
  expr: increase(kube_proxy_sync_proxy_rules_errors_total[5m]) > 3
  labels:
    severity: critical
```

### 4. Documentation Standard
```markdown
## Node Firewall Policy
- All custom rules MUST:
  - Use dedicated chains (`-N CUSTOM-NAME`)
  - Never modify `KUBE-*` chains directly  
  - Be version-controlled in `/etc/iptables.rules`
```

---

**Key Metrics to Monitor**:  
- `kube_proxy_sync_proxy_rules_duration_seconds`  
- `node_netstat_IpExt_OutNoRoutes`  
- `kube_proxy_network_programming_duration_seconds`  

**Debugging Tools**:  
```sh
# Compare expected vs actual rules
diff <(kube-proxy --cleanup) <(iptables-save)

# Rule visualization
iptables -t nat -L -v --line-numbers
```

**kube-proxy Health Checks**:  
```sh
# Verify rule sync status
curl -s http://localhost:10249/proxyMode
# Should return "iptables" with no errors
``` 

**Backup Procedure**:  
```sh
# Daily iptables backup
iptables-save > /etc/iptables.rules.$(date +%F)
```
---
---
# 📘 Scenario #12: Stuck CSR Requests Blocking New Node Joins

**Category**: Cluster Authentication  
**Environment**: Kubernetes v1.20, kubeadm cluster  
**Impact**: Node provisioning freeze, scaling operations failed  

---

## Scenario Summary  
A disabled CSR approval controller caused a backlog of 500+ unapproved certificate signing requests, preventing new nodes from joining the cluster and existing nodes from renewing certificates.  

---

## What Happened  
- **Security patch misstep**:  
  - `--cluster-signing-cert-file` flag was removed during CIS hardening  
  - CSR approval controller silently stopped processing requests  
- **Symptoms emerged**:  
  - New nodes stuck at `[kubelet] Waiting for server to sign certificate`  
  - Existing nodes began failing health checks as certificates expired  
  - `kubectl get nodes` showed `NotReady` for recently added nodes  
- **Certificate chain reaction**:  
  - Kubelet client certs (default 1y expiry) started expiring en masse  
  - Cluster-autoscaler couldn't provision worker nodes  

---

## Diagnosis Steps  

### 1. Check CSR backlog:
```sh
kubectl get csr -o wide | grep -c Pending
# Output showed 527 pending requests
```

### 2. Inspect kubelet logs:
```sh
journalctl -u kubelet --no-pager | grep -A10 "certificate"
# "Failed to request signed certificate: timed out waiting for CSR to be signed"
```

### 3. Verify controller status:
```sh
kubectl -n kube-system get pod -l component=kube-controller-manager \
  -o jsonpath='{.items[0].spec.containers[0].command}' | jq
# Missing --cluster-signing-cert-file flag
```

### 4. Check certificate expiry:
```sh
openssl x509 -enddate -noout -in /var/lib/kubelet/pki/kubelet-client-current.pem
# "notAfter=Apr 12 13:32:42 2023 GMT" (expired)
```

---

## Root Cause  
**Broken auto-approval flow**:  
1. Security hardening disabled certificate signing controller  
2. No monitoring for CSR approval latency  
3. Certificate expiry wave compounded the problem  

## Fix/Workaround  

### Emergency Recovery:
```sh
# Bulk approve node client CSRs
kubectl get csr -o name | grep 'node-client' | xargs kubectl certificate approve

# Approve all pending kubelet-serving CSRs
kubectl get csr -o json | \
  jq -r '.items[] | select(.status == {}) | .metadata.name' | \
  xargs kubectl certificate approve
```

### Controller Restoration:
```yaml
# /etc/kubernetes/manifests/kube-controller-manager.yaml
spec:
  containers:
  - command:
    - kube-controller-manager
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,csrapproving  # Explicitly enable
```

---

---

## Lessons Learned  
⚠️ **CSRs are cluster lifeblood**: Controls node auth and TLS rotations  
⚠️ **Silent failures**: Controllers fail open with no alerts  
⚠️ **Certificate expiry waves**: Synchronized issuance causes time-bomb effects  

---

## Prevention Framework  

### 1. Automated Approval Policies
```yaml
# CSR Approver ClusterRole (ensure it exists)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/nodeclient"]
  verbs: ["create"]
```

### 2. Monitoring
```yaml
# Critical Prometheus alerts
- alert: PendingCSRsHigh
  expr: count(kube_certificatesigningrequest_status_condition{condition="Approved",status="false"}) > 10
  for: 5m
  labels:
    severity: critical

- alert: CertificateExpirySoon
  expr: kubelet_certificate_manager_client_expiration_seconds < 86400*7  # 7 days
```

### 3. Automation Safeguards
```sh
# Pre-flight check for new nodes
kubeadm join --dry-run | grep -q "certificate signing request" || \
  { echo "CSR flow broken"; exit 1; }
```

### 4. Certificate Management
```yaml
# KubeletConfiguration (rotate more frequently)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
rotateCertificates: true
certificateRotationStrategy:
  expirationSeconds: 86400*30  # 30-day certs
```

---

**Key Metrics to Monitor**:  
- `kube_certificatesigningrequest_status_condition`  
- `kubelet_certificate_manager_client_expiration_seconds`  
- `controller_manager_ttl_after_finished_seconds`  

**Debugging Tools**:  
```sh
# Inspect CSR details
kubectl get csr -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.username}{"\t"}{.status.conditions[*].type}{"\n"}{end}'

# Verify CA certificate chain
openssl verify -CAfile /etc/kubernetes/pki/ca.crt /var/lib/kubelet/pki/kubelet-client-current.pem
```

**Renewal Procedure**:  
```sh
# Force certificate renewal (on node)
systemctl restart kubelet
rm /var/lib/kubelet/pki/kubelet-client-*
```

---
---
# 📘 Scenario #13: Failed Cluster Upgrade Due to Unready Static Pods

**Category**: Control Plane Maintenance  
**Environment**: Kubernetes v1.21 → v1.23 upgrade, kubeadm  
**Impact**: Upgrade rollback required, 2-hour control plane outage  

---

## Scenario Summary  
A malformed etcd static pod manifest prevented the control plane from coming up during a minor version upgrade, requiring manual intervention and causing extended downtime.  

---

## What Happened  
- **Pre-upgrade change**:  
  - Admin modified `/etc/kubernetes/manifests/etcd.yaml` to add new volume  
  - Introduced typo in `volumeMounts` (`/var/lib/etcd` vs `/var/lib/etc`)  
- **Upgrade failure**:  
  - `kubeadm upgrade apply` hung at `[control-plane] Creating static Pod manifests`  
  - `kubelet` logged `Failed to create pod sandbox: invalid volume mount path`  
  - API server became unreachable after old etcd pod terminated  
- **Cascading effects**:  
  - Worker nodes marked control plane `NotReady` after 5m  
  - Scheduler stopped assigning new pods  

---

## Diagnosis Steps  

### 1. Verify static pod status:
```sh
# On control plane node:
crictl pods --name etcd
# Showed "NotReady" state
```

### 2. Inspect kubelet logs:
```sh
journalctl -u kubelet -n 50 --no-pager | grep -A10 "static pod"
# Error: "invalid volume mount path: /var/lib/etc"
```

### 3. Validate manifest syntax:
```sh
yamllint /etc/kubernetes/manifests/etcd.yaml
# Passed (syntax valid but path incorrect)
```

### 4. Check etcd container:
```sh
crictl inspect $(crictl ps -a --name etcd -q) | jq '.status.reason'
# "CreateContainerError"
```

---

## Root Cause  
**Manifest validation gap**:  
1. **No pre-flight validation** of static pod changes  
2. **Case-sensitive path** typo went undetected  
3. **kubelet retry logic** didn't surface error clearly  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Restore from backup manifest
cp /etc/kubernetes/manifests/etcd.yaml.bak /etc/kubernetes/manifests/etcd.yaml

# 2. Force kubelet reload
systemctl restart kubelet

# 3. Verify etcd recovery
crictl logs $(crictl ps --name etcd -q)
```


### Complete Upgrade:
```sh
kubeadm upgrade apply v1.23.5 --force
```

## Lessons Learned  
⚠️ **Static pods are fragile**: No admission controllers validate them  
⚠️ **kubelet is silent**: Errors only appear in logs, not `kubectl`  
⚠️ **Control plane domino effect**: etcd failure → API failure → Cluster failure  

---

## Prevention Framework  

### 1. Manifest Management
```sh
# Pre-change validation script
validate_manifest() {
  yamllint $1 && \
  kubeval --strict $1 --schema-location https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master
}
```

### 2. Change Control
```yaml
# Ansible pre-upgrade checklist
- name: Backup static manifests
  copy:
    src: "/etc/kubernetes/manifests/{{ item }}"
    dest: "/etc/kubernetes/manifests/{{ item }}.bak-{{ ansible_date_time.iso8601 }}"
  loop:
    - etcd.yaml
    - kube-apiserver.yaml
```

### 3. Monitoring
```yaml
# Prometheus alerts for static pods
- alert: StaticPodNotReady
  expr: time() - kube_pod_start_time{created_by_kind="Node"} > 300
  for: 2m
  labels:
    severity: critical
```

### 4. Upgrade Safeguards
```sh
# kubeadm pre-flight check
kubeadm upgrade diff --config=kubeadm-config.yaml
```

---

**Key Metrics to Monitor**:  
- `kube_pod_status_ready{created_by_kind="Node"}`  
- `kubelet_running_pods`  
- `etcd_server_has_leader`  

**Debugging Tools**:  
```sh
# Live static pod inspection
crictl inspectp $(crictl pods --name etcd -q) | jq '.status.conditions'

# Compare manifests
diff -u /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/manifests/etcd.yaml.bak
```

**Backup Procedure**:  
```sh
# Daily manifest backup
tar czf /backups/k8s-manifests-$(date +%F).tgz /etc/kubernetes/manifests
```

**Static Pod Linter**:  
```yaml
# Example etcd manifest validation rules
checks:
  - id: etcd-volume-mounts
    severity: ERROR
    message: "ETCD data volume must mount to /var/lib/etcd"
    paths:
      - /etc/kubernetes/manifests/etcd.yaml
    assert:
      all:
        - exists: $.spec.containers[0].volumeMounts[?(@.mountPath == '/var/lib/etcd')]
```

---
---

# 📘 Scenario #14: Uncontrolled Logs Filled Disk on All Nodes

**Category**: Node Resource Management  
**Environment**: Kubernetes v1.24, AWS EKS, containerd runtime  
**Impact**: Cluster-wide disk pressure, pod evictions, and application failures  

---

## Scenario Summary  
A misconfigured debug flag in a production pod caused log explosions (100+ MB/sec), filling up `/var/log` across all worker nodes and triggering `DiskPressure` evictions.

---

## What Happened  
- **Accidental debug deployment**:  
  - `LOG_LEVEL=TRACE` committed to production configmap  
  - Deployed to 50+ replicas  
- **Storage symptoms**:  
  - `/var/log` reached 100% capacity in under 2 hours  
  - `kubelet` logged `No space left on device` errors  
  - Containerd became unresponsive (`failed to create task: write /var/log/...`)  
- **Cascading effects**:  
  - Node `NotReady` status due to failed health checks  
  - Cluster autoscaler spun up new nodes (which also filled rapidly)  

---

## Diagnosis Steps  

### 1. Identify disk usage:
```sh
kubectl debug node/<node> -it --image=alpine -- df -h /var/log
# Output: 100% used
```

### 2. Locate log offenders:
```sh
kubectl debug node/<node> -it --image=alpine -- \
  du -ah /var/log/containers/ | sort -rh | head -20
```

### 3. Verify pod logs:
```sh
kubectl logs <offending-pod> --tail=100 | grep -c "TRACE"
# Output: 100/100 lines at TRACE level
```

### 4. Check log rotation config:
```sh
kubectl debug node/<node> -it --image=alpine -- \
  cat /etc/docker/daemon.json | jq '.["log-opts"]'
# Showed missing size limits
```

---

## Root Cause  
**Unbounded logging**:  
1. No log rotation or size limits in container runtime config  
2. CI/CD allowed `TRACE` level in production  
3. Missing pod-level log quotas  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Free up space (on affected nodes)
kubectl debug node/<node> -it --image=alpine -- \
  truncate -s 0 /var/log/containers/*_<namespace>_<pod-name>*.log

# 2. Restart containerd
kubectl debug node/<node> -it --image=alpine -- \
  systemctl restart containerd

# 3. Scale down offender
kubectl scale deploy <debug-deployment> --replicas=0
```

### Long-term Solution:
```json
// /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd]
  disable_hugetlb_controller = false
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
  [plugins."io.containerd.grpc.v1.cri".containerd.logs]
    max_size = "100MB"
    max_files = 3
```

---
## Lessons Learned  
⚠️ **Logs are unmanaged resources**: Can consume 100% disk if unchecked  
⚠️ **Debug levels are dangerous**: `TRACE`/`DEBUG` must never reach production  
⚠️ **Node storage is shared**: One pod can take down entire node  

---

## Prevention Framework  

### 1. Runtime Log Policies
```yaml
# EKS AMI bootstrap configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
containerLogMaxSize: "50Mi"
containerLogMaxFiles: 3
``

### 2. Deployment Safeguards
```yaml
# OPA/Gatekeeper policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedLogLevels
metadata:
  name: deny-trace-logs
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    forbiddenLevels: ["TRACE", "DEBUG"]
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: NodeLogDiskFull
  expr: node_filesystem_avail_bytes{mountpoint="/var/log"} / node_filesystem_size_bytes{mountpoint="/var/log"} < 0.2
  for: 5m
  labels:
    severity: critical
```

### 4. CI/CD Validation
```sh
# Pre-deployment check
if grep -q "LOG_LEVEL=TRACE" $MANIFEST; then
  echo "ERROR: TRACE logging prohibited in production"
  exit 1
fi
```

---

**Key Metrics to Monitor**:  
- `container_fs_usage_bytes`  
- `node_filesystem_avail_bytes{mountpoint="/var/log"}`  
- `kubelet_evictions` by `DiskPressure`  

**Debugging Tools**:  
```sh
# Live log rate monitoring
kubectl exec <pod> -- sh -c "tail -f /proc/1/fd/1 | pv -lbtr >/dev/null"

# Per-pod log size
kubectl get pods -o json | jq '.items[] | {name:.metadata.name, logs:.spec.containers[].resources.limits."logging-size"}'
```

**Log Rotation Policy**:  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-limited
spec:
  containers:
  - name: app
    resources:
      limits:
        logging-size: "100Mi"
```

---
---
# 📘 Scenario #15: Node Drain Fails Due to PodDisruptionBudget Deadlock

**Category**: Cluster Reliability  
**Environment**: Kubernetes v1.21, Production Cluster with HPA  
**Impact**: Unplanned maintenance delays, failed node rotations  

---