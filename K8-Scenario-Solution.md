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

## Scenario Summary  
A `PodDisruptionBudget` (PDB) deadlock prevented node drainage when a deployment's `minAvailable` requirement exceeded available replicas during maintenance operations.

---

## What Happened  
- **Maintenance trigger**:  
  - Node required emergency patching (CVE-2021-44228)  
  - `kubectl drain` command hung indefinitely  
- **PDB conflict**:  
  - Deployment had `replicas: 2` with PDB `minAvailable: 2`  
  - Zero allowed disruptions (`kubectl get pdb` showed `ALLOWED-DISRUPTIONS: 0`) 
- **System behavior**:  
  - Drain operation timed out after 1h (`error when evicting pod: cannot evict as it would violate the pod's disruption budget`)  
  - Cluster autoscaler refused to scale up (CPU metrics below threshold)  

---

## Diagnosis Steps  

### 1. Check PDB status:
```sh
kubectl get pdb -A -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,MIN-AVAILABLE:.spec.minAvailable,PODS:.status.currentHealthy,ALLOWED:.status.disruptionsAllowed"
```

### 2. Verify deployment scale:
```sh
kubectl get deploy -n <namespace> -o jsonpath='{.items[*].spec.replicas}'
# Output: 2
```

### 3. Inspect drain status:
```sh
kubectl get events --field-selector involvedObject.kind=Pod --sort-by=.lastTimestamp
# Showed repeated eviction failures
```

### 4. Check HPA constraints:
```sh
kubectl get hpa -n <namespace> -o yaml | yq '.items[].spec.minReplicas'
# Output: 2 (locked scale)
```

---

## Root Cause  
**Availability guarantee paradox**:  
1. PDB `minAvailable` == replica count → Zero eviction headroom  
2. HPA prevented scale-up during low traffic periods  
3. No coordination between PDB definitions and maintenance procedures  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# Option 1: Temporarily relax PDB (if SLA allows)
kubectl patch pdb <name> -p '{"spec":{"minAvailable":1}}'

# Option 2: Force scale-up first
kubectl scale deploy <name> --replicas=3
```

### Complete Drain:
```sh
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

### Post-Maintenance:
```sh
# Restore original PDB
kubectl patch pdb <name> -p '{"spec":{"minAvailable":2}}'

# Optional scale-down
kubectl scale deploy <name> --replicas=2
```

---

## Lessons Learned  
⚠️ **PDBs can create hard locks**: Exact `minAvailable` matches are dangerous  
⚠️ **HPA interactions matter**: Minimum replicas must exceed PDB requirements  
⚠️ **Maintenance needs headroom**: Always design for N+1 availability during operations  

---

## Prevention Framework  

### 1. Validation Webhooks
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sValidPodDisruptionBudget
metadata:
  name: pdb-min-available-check
spec:
  match:
    kinds:
    - apiGroups: ["policy"]
      kinds: ["PodDisruptionBudget"]
  parameters:
    minReplicaBuffer: 1  # Require minAvailable < replicas
```

### 2. CI/CD Checks
```sh
# Pre-deployment validation script
check_pdb() {
  replicas=$(kubectl get deploy $1 -o jsonpath='{.spec.replicas}')
  minAvailable=$(kubectl get pdb $1 -o jsonpath='{.spec.minAvailable}')
  [ $replicas -gt $minAvailable ] || {
    echo "ERROR: PDB minAvailable >= replicas"
    exit 1
  }
}
```

### 3. Monitoring
```yaml
# Critical Prometheus alerts
- alert: PDBBlocksEvictions
  expr: kube_poddisruptionbudget_status_disruptions_allowed == 0
  for: 15m
  labels:
    severity: warning
  annotations:
    description: PDB {{ $labels.namespace }}/{{ $labels.poddisruptionbudget }} has zero allowed disruptions
```

### 4. Drain Automation
```yaml
# Ansible playbook snippet
- name: Ensure drain capacity
  k8s:
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: "{{ item }}"
        namespace: production
      spec:
        replicas: "{{ deployment_replicas | int + 1 }}"
  when: maintenance_mode
```

---

**Key Metrics to Monitor**:  
- `kube_poddisruptionbudget_status_disruptions_allowed`  
- `kube_deployment_spec_replicas` vs `kube_poddisruptionbudget_spec_min_available`  
- `kube_node_status_condition{condition="Ready"}` during maintenance  

**Debugging Tools**:  
```sh
# Simulate drain impact
kubectl drain <node> --dry-run=server

# Check PDB calculations
kubectl get pdb -o json | jq '.items[] | {name:.metadata.name, min:.spec.minAvailable, healthy:.status.currentHealthy, allowed:.status.disruptionsAllowed}'
```

**PDB Design Guidelines**:  

1. Always set `minAvailable` ≤ (replicas - 1)  
2. For critical systems, use percentage-based PDBs (`minAvailable: 50%`)  
3. Annotate PDBs with maintenance instructions:  
   ```yaml
   annotations:
     cluster/ops-maintenance-protocol: "Scale to 3+ replicas before drain"
   ```

---
---
# 📘 Scenario #16: CrashLoop of Kube-Controller-Manager on Boot

**Category**: Control Plane Stability  
**Environment**: Kubernetes v1.23, Self-hosted Control Plane  
**Impact**: Cluster-wide controller failures (deployments, services stalled)  

---

## Scenario Summary  
The `kube-controller-manager` entered a crashloop after a cluster upgrade due to an obsolete admission plugin in its startup configuration, paralyzing core cluster operations.

---

## What Happened  
- **Post-upgrade failure**:  
  - Control plane pods restarted after `kubeadm upgrade` to v1.23  
  - `kube-controller-manager` crashed immediately with `unknown admission plugin` error  
- **System impact**:  
  - No new deployments could be created (`No controllers available to schedule`)  
  - Existing deployments stopped scaling (`Failed to list *v1.ReplicaSet`)  
- **Configuration mismatch**:  
  - Legacy `--enable-admission-plugins=NamespaceLifecycle,InitialResources` flag  
  - `InitialResources` plugin removed in Kubernetes 1.22  

---

## Diagnosis Steps  

### 1. Check controller-manager status:
```sh
kubectl -n kube-system get pod -l component=kube-controller-manager
# Showed CrashLoopBackOff
```

### 2. Inspect logs:
```sh
kubectl -n kube-system logs --tail=50 -l component=kube-controller-manager
# Error: "unknown admission plugin \"InitialResources\""
```

### 3. Verify version compatibility:
```sh
kube-controller-manager --version | grep -q "v1.23" || \
  echo "Version mismatch"
```

### 4. Check startup flags:
```sh
ps aux | grep kube-controller-manager | grep -o "\-\-enable\-admission\-plugins=[^ ]*"
# Output: --enable-admission-plugins=NamespaceLifecycle,InitialResources
```

## Root Cause  
**Breaking change unhandled**:  
1. Admission plugin `InitialResources` removed in v1.22 (unnoticed during upgrade planning)  
2. No pre-flight validation of controller-manager flags  
3. Static pod manifest not version-controlled  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Edit static manifest
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
# Remove InitialResources from --enable-admission-plugins

# 2. Force kubelet to reload
systemctl restart kubelet

# 3. Verify recovery
kubectl -n kube-system wait pod -l component=kube-controller-manager --for=condition=Ready
```

### Permanent Solution:
```yaml
# Updated admission control configuration
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --enable-admission-plugins=NamespaceLifecycle,ServiceAccount
    - --disable-admission-plugins=InitialResources  # Explicit disable
```

---

## Lessons Learned  
⚠️ **Admission plugins are version-locked**: Removed plugins cause hard failures  
⚠️ **Static pods need upgrade testing**: Cannot rely on `kubeadm` to fix all configs  
⚠️ **Controller-manager is critical**: Crashes halt deployments, services, and more  

---

## Prevention Framework  

### 1. Version Upgrade Checklist
```markdown
1. [ ] Review deprecated API removals in changelog  
2. [ ] Audit all `--enable-admission-plugins` flags  
3. [ ] Test control plane components in staging  
```

### 2. Configuration Validation
```sh
# Pre-upgrade compatibility check
kube-controller-manager --enable-admission-plugins=... --dry-run | grep -q "unknown" && \
  echo "Invalid plugins detected"
```

### 3. GitOps for Control Plane
```yaml
# ArgoCD Application for kube-system
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: control-plane
spec:
  source:
    path: kubernetes/manifests
    repoURL: git@github.com:org/cluster-config.git
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 4. Monitoring
```yaml
# Prometheus alert for controller health
- alert: ControllerManagerDown
  expr: absent(up{job="kube-controller-manager"} == 1)
  for: 1m
  labels:
    severity: critical
```

---

**Key Metrics to Monitor**:  
- `controller_manager_running_controllers`  
- `workqueue_depth` (per controller)  
- `rest_client_requests_total{code!~"2.."}`  

**Debugging Tools**:  
```sh
# Check available admission plugins
kube-controller-manager --help | grep "enable-admission-plugins"

# Diff current vs expected manifests
diff -u /etc/kubernetes/manifests/kube-controller-manager.yaml /gitops/manifests/kube-controller-manager.yaml
```

**Admission Plugin Policy**:  
```yaml
# Recommended v1.23+ plugins
enable-admission-plugins: |
  NamespaceLifecycle,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,
  ResourceQuota,Priority,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
```

---
---

# 📘 Scenario #17: Inconsistent Cluster State After Partial Backup Restore

**Category**: Disaster Recovery  
**Environment**: Kubernetes v1.24, Velero with etcd snapshots  
**Impact**: Production outage lasting 4+ hours due to incomplete state restoration  

---

## Scenario Summary  
A partial etcd restore operation created a "zombie cluster" state where API objects existed without their dependent resources (PVCs, Secrets), causing widespread pod failures.

---

## What Happened  
- **Disaster recovery scenario**:  
  - etcd corruption required restore from 6-hour-old Velero snapshot  
  - Only etcd data was restored (excluded PV snapshots and external Secrets)  
- **Post-restore symptoms**:  
  - 60% of pods stuck in `CreateContainerError`  
  - Persistent volume claims showed `<none>` in `kubectl get pvc -A`  
  - CSI driver logs reported `volume not found` errors  
- **Dependency graph breaks**:  
  - Deployments referenced non-existent PVCs  
  - ServiceAccounts lacked corresponding Secrets  

---

## Diagnosis Steps  

### 1. Verify resource consistency:
```sh
# Check for orphaned objects
kubectl get deployments,statefulsets -A -o json | jq -r '.items[] | select(.status.replicas != .status.readyReplicas) | .metadata.name'

# Compare object counts
velero backup describe $BACKUP --details | grep -c "Resource"
kubectl api-resources --verbs=list -o name | xargs -n1 kubectl get -A --ignore-not-found | wc -l
```

### 2. Inspect storage state:
```sh
kubectl get pv,pvc -A --no-headers | grep -v Bound
# Showed unbound claims
```

### 3. Check secret dependencies:
```sh
kubectl get secrets -A | grep -E "(default-token|registry-key)"
# Missing critical secrets
```

### 4. Audit Velero restore logs:
```sh
velero restore logs $RESTORE | grep -i "skipped\|failed"
# Revealed excluded resources
```

---

## Root Cause  
**Incomplete backup strategy**:  
1. Velero configured without `--snapshot-volumes=true`  
2. No coordination between etcd backups and CSI snapshots  
3. External secrets (Vault) not included in backup scope  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# 1. Recreate critical secrets from backup
kubectl create secret generic registry-key \
--from-file=./backups/secrets/registry-key.json

# 2. Provision replacement PVs
velero restore create --from-backup $PVC_BACKUP --include-resources persistentvolumeclaims
```

### Full Restoration:
```sh
# 3. Redeploy applications with corrected dependencies
kubectl rollout restart deploy -n production
```

---

## Lessons Learned  
⚠️ **etcd is only part of the picture**: Application state spans multiple systems  
⚠️ **Dependencies matter**: Objects without their dependencies create "zombie" resources  
⚠️ **Silent exclusions**: Backup tools often skip resources by default  

---

## Prevention Framework  

### 1. Holistic Backup Policy
```yaml
# velero-backup.yaml
apiVersion: velero.io/v1
kind: Backup
spec:
  includedResources:
  - '*'
  excludedResources:
  - nodes,events,events.events.k8s.io
  snapshotVolumes: true
  ttl: 720h
  hooks:
    resources:
    - name: pre-vault-backup
      pre:
        - exec:
            container: vault-agent
            command: ["/bin/sh", "-c", "vault operator raft snapshot save /backups/vault.snap"]
```

### 2. Restore Validation
```sh
# Post-restore checklist
validate_restore() {
  kubectl wait --for=condition=Ready pods --all --timeout=10m
  kubectl get pvc -A | grep -v Bound && echo "Unbound PVCs detected"
}
```

### 3. Backup Monitoring
```yaml
# Prometheus alerts
- alert: IncompleteBackup
  expr: velero_backup_success_total == 0 or velero_volume_snapshot_failure_total > 0
  for: 15m
  labels:
    severity: critical
```

### 4. Disaster Recovery Drills
```markdown
Quarterly Test Plan:
1. [ ] Simulate etcd corruption  
2. [ ] Restore from Velero + CSI snapshots  
3. [ ] Validate application connectivity  
4. [ ] Measure time-to-recovery (TTR)  
```

---

**Key Resources to Backup**:  
- **etd**: All Kubernetes objects  
- **CSI**: Persistent Volumes  
- **External**: Secrets (Vault/HashiCorp), Container Images  
- **Configuration**: CNI, CSI, Ingress controllers  


**Restore Verification Tools**:  
```sh
# Check object consistency
kubectl get all,cm,secret,pvc -A --no-headers | wc -l
# Compare with backup
velero backup describe $BACKUP --details | grep "Resource"

# Validate storage
kubectl exec -it test-pod -- df -h /data
```
**Velero Best Practices**:  
```yaml
# Schedule with hooks
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-full
spec:
  schedule: "@daily"
  template:
    hooks:
      resources:
      - name: freeze-db
        pre:
        - exec:
            container: db
            command: ["/bin/sh", "-c", "flush tables with read lock"]
        post:
        - exec:
            container: db
            command: ["/bin/sh", "-c", "unlock tables"]
```
---
---
# 📘 Scenario #18: kubelet Unable to Pull Images Due to Proxy Misconfig

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.25, Corporate Proxy Environment  
**Impact**: Deployment failures across 200+ nodes, 4-hour service disruption  

---

## Scenario Summary  
A missing `NO_PROXY` configuration caused kubelet to route all container image pulls through an external proxy, breaking internal registry access and cluster DNS resolution.

---

## What Happened  
- **Proxy enforcement rollout**:  
  - Corporate policy mandated HTTP_PROXY for all outbound traffic  
  - Kubelet config updated with `HTTP_PROXY=http://proxy.corp:3128`  
- **Failure symptoms**:  
  - New pods stuck in `ImagePullBackOff`  
  - `kubelet` logs showed `Failed to pull image: context deadline exceeded`  
  - Internal registry metrics showed 100% connection timeouts  
- **Network analysis**:  
  - Internal `10.0.100.0/24` traffic was being routed externally  
  - CoreDNS queries to `kubernetes.default.svc` timed out  

---

## Diagnosis Steps  

### 1. Verify image pull failures:
```sh
kubectl get pods -A -o wide | grep -E 'ImagePullBackOff|ErrImagePull'
```

### 2. Inspect kubelet proxy config:
```sh
systemctl show kubelet --property=Environment --no-pager
# Output showed HTTP_PROXY set but NO_PROXY missing
```

### 3. Test cluster DNS through proxy:
```sh
kubectl run -it --rm netcheck --image=busybox -- \
  sh -c "wget -O- http://kubernetes.default.svc --proxy=on"
# Failed with 504 Gateway Timeout
```

### 4. Check registry access:
```sh
kubectl debug node/<node> -it --image=alpine -- \
  curl -x $HTTP_PROXY https://registry.internal.corp/v2/
# Returned 407 Proxy Authentication Required
```

---

## Root Cause  
**Proxy overreach**:  
1. Missing `NO_PROXY` exclusions for:  
   - Cluster CIDR (`10.0.0.0/8`)  
   - Service CIDR (`192.168.0.0/16`)  
   - Internal domains (`*.svc.cluster.local`)  
2. Proxy server blocked internal IP ranges  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Edit kubelet service config
sudo mkdir -p /etc/systemd/system/kubelet.service.d
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.corp:3128"
Environment="HTTPS_PROXY=http://proxy.corp:3128"
Environment="NO_PROXY=10.0.0.0/8,192.168.0.0/16,.svc,.cluster.local,localhost,127.0.0.1"
EOF

# 2. Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Cluster-wide Update:
```yaml
# Ansible playbook snippet
- name: Configure kubelet proxy
  template:
    src: proxy.conf.j2
    dest: /etc/systemd/system/kubelet.service.d/proxy.conf
    owner: root
    group: root
  vars:
    no_proxy_ranges:
      - "10.0.0.0/8"
      - "{{ kube_service_addresses }}"
      - ".{{ cluster_domain }}"
```

---


## Lessons Learned  
⚠️ **Proxies break cluster networking**: Must exclude all internal traffic  
⚠️ **Kubelet has special requirements**: Needs direct access to DNS + registry  
⚠️ **Corporate policies need adaptation**: Kubernetes isn't a standard workload  

---

## Prevention Framework  

### 1. Proxy Configuration Template
```ini
# /etc/systemd/system/kubelet.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.corp:3128"
Environment="HTTPS_PROXY=http://proxy.corp:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,.svc,.cluster.local,.internal.corp"
```

### 2. Validation Checks
```sh
# Pre-apply proxy test
validate_proxy() {
  kubectl run -it --rm proxy-test --image=alpine -- \
    sh -c "env | grep -E 'NO_PROXY|cluster.local' || exit 1"
}
```


### 3. Monitoring
```yaml
# Prometheus alerts
- alert: ProxyImagePullFailures
  expr: increase(kubelet_image_pull_failures_total[1h]) > 10
  labels:
    severity: critical
  annotations:
    summary: "Image pulls failing through proxy (instance {{ $labels.instance }})"
```

### 4. Documentation Standards

## Corporate Proxy Requirements for Kubernetes
- **Always exclude**:
  - Cluster CIDR (e.g., `10.0.0.0/8`)
  - Service CIDR (e.g., `192.168.0.0/16`)
  - DNS suffixes (`.svc`, `.cluster.local`)

- **Test before rollout**:
  ```sh
  kubectl run proxy-test --image=busybox -- \
    wget -O- http://kubernetes.default.svc
  ```


---

**Key NO_PROXY Entries for Kubernetes**:  
- Cluster Pod CIDR  
- Service CIDR  
- `.svc`, `.svc.cluster.local`  
- Internal registry domains  
- Node IP ranges 

**Debugging Tools**:  
```sh
# Verify current proxy settings
kubectl debug node/<node> -it --image=alpine -- env | grep -i proxy

# Test registry access
kubectl run -it --rm registry-test --image=alpine -- \
  curl -v http://registry.internal.corp/v2/_catalog
```

**Rollback Procedure**:  
```sh
# Emergency proxy disable
sudo rm /etc/systemd/system/kubelet.service.d/proxy.conf
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---
---
# 📘 Scenario #19: Multiple Nodes Marked Unreachable Due to Flaky Network Interface

**Category**: Infrastructure Reliability  
**Environment**: Kubernetes v1.22, Bare-metal, Bonded NICs  
**Impact**: Intermittent node failures, pod evictions, and workload disruptions  

---

## Scenario Summary  
A flapping network interface caused multiple nodes to oscillate between `Ready` and `NotReady` states, triggering unnecessary pod evictions and workload rescheduling.

---

## What Happened  
- **Network instability**:  
  - Nodes randomly reported `NotReady` for 30-60 second intervals  
  - `kube-controller-manager` logs showed frequent `NodeNotReady` events
- **Observed symptoms**:  
  - Pods evicted with `NodeNetworkUnavailable` reason  
  - Cluster autoscaler provisioned unnecessary replacement nodes  
  - Storage systems (CSI) timed out during network drops  
- **Root discovery**:  
  - Switch logs showed `port up/down` events every 2-3 minutes  
  - `ethtool` reported `link flaps: 142` on affected nodes  

---

## Diagnosis Steps  

### 1. Check node status history:
```sh
kubectl get nodes -o wide --watch | tee node-status.log
# Showed flapping between Ready/NotReady
```

### 2. Inspect network interfaces:
```sh
ethtool eno1 | grep -A5 'Link detected'
# Reported intermittent link drops
```

### 3. Analyze kernel logs:
```sh
dmesg -T | grep -i 'link down\|nic'
# Revealed NIC resets
```

### 4. Verify switch port status:
```sh
ssh switch01 show interface ethernet 1/0/24 | include 'error|flap'
# Output: "Last link flapped: 00:02:15 ago"
```

---

## Root Cause  
**Physical layer instability**:  
1. **Faulty SFP+ transceiver** causing signal degradation  
2. **Loose fiber connection** in one port of the bonded interface  
3. **No LACP fallback** configured for single-port failures  

---

## Fix/Workaround  

### Immediate Actions:
```sh
# 1. Disable problematic switch port
ssh switch01 configure terminal
interface ethernet 1/0/24
shutdown

# 2. Drain affected nodes
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

### Permanent Solution:
```sh
# 1. Replace hardware (transceiver + cable)
# 2. Configure active-backup bonding:
nmcli con add type bond con-name bond0 ifname bond0 \
  mode active-backup miimon 100 primary eth0
nmcli con add type bond-slave ifname eth0 master bond0
nmcli con add type bond-slave ifname eth1 master bond0
```

---

## Lessons Learned  
⚠️ **Kubernetes amplifies physical issues**: Short blips cause pod evictions  
⚠️ **Bonding != redundancy**: Must test failover scenarios  
⚠️ **Hardware fails silently**: Requires active monitoring  

---

## Prevention Framework  

### 1. Network Bonding Configuration
```yaml
# NetworkManager bond example
connections:
- name: bond0
  type: bond
  interface-name: bond0
  bond:
    options:
      mode: "802.3ad"
      miimon: "100"
      lacp-rate: "fast"
```

### 2. Monitoring
```yaml
# Prometheus alerts
- alert: NodeNetworkFlapping
  expr: changes(kube_node_status_condition{condition="Ready",status="true"}[15m]) > 3
  for: 5m
  labels:
    severity: warning
    
- alert: NICErrorsHigh
  expr: node_network_up == 0 or rate(node_network_transmit_errs_total[5m]) > 5
  labels:
    severity: critical
```

### 3. Hardware Checks
```sh
# Daily NIC health check (via CronJob)
ethtool -S eth0 | grep -E 'err|drop'
smartctl -H /dev/nvme0
```

### 4. Documentation

## Bonding Best Practices
- **Mode**: 802.3ad (LACP) for switches, active-backup for redundancy  
- **Monitoring**: Track `carrier_changes` and `link_flaps`  
- **Testing**:  
  ```sh
  ip link set eth0 down && sleep 30 && ip link set eth0 up
  ```

---


**Key Metrics to Monitor**:  
- `node_network_carrier_changes`  
- `node_network_up`  
- `kube_node_status_condition{condition="NetworkUnavailable"}`  

**Debugging Tools**:  
```sh
# Live packet capture (tcpdump)
kubectl debug node/<node> -it --image=nicolaka/netshoot -- \
  tcpdump -i bond0 -w /host/tmp/bond0.pcap

# Bonding status
cat /proc/net/bonding/bond0
```

**Switch Configuration**:  
```cisco
interface Port-channel1
  description Kubernetes-Bond
  switchport mode trunk
  channel-group 1 mode active
  lacp fast-switchover
```

**Rollback Procedure**:  
```sh
# Emergency single-interface fallback
ip link set bond0 down
ip link set eth0 up
ip addr add <NODE_IP>/24 dev eth0
```
---
---


# 📘 Scenario #20: Node Labels Accidentally Overwritten by DaemonSet

**Category**: Cluster Configuration  
**Environment**: Kubernetes v1.24, Label Management DaemonSet  
**Impact**: Critical GPU workloads unscheduled for 3+ hours  

---

## Scenario Summary  
A well-intentioned but overly aggressive DaemonSet overwrote critical node labels (`gpu=true`, `storage=ssd`), disrupting scheduler decisions and causing workload placement failures.

---

## What Happened  
- **Automated label deployment**:  
  - DaemonSet designed to standardize zone labels (`zone=us-east-1a`)  
  - Used `kubectl label --overwrite` instead of strategic merge  
- **Immediate symptoms**:  
  - GPU-requiring pods stuck in `Pending` (`0/3 nodes available: 3 node(s) didn't match Pod's node affinity`)  
  - Storage-sensitive workloads scheduled to HDD nodes  
- **Configuration drift**:  
  - 28 nodes lost 5+ custom labels each  
  - Scheduler metrics showed `FailedScheduling` events spiked 400%  

---

## Diagnosis Steps  

### 1. Identify scheduling failures:
```sh
kubectl get events --field-selector reason=FailedScheduling -A
```

### 2. Compare node labels pre/post incident:
```sh
# Get current state
kubectl get nodes -L gpu,storage,zone

# Compare with backup (if available)
diff <(kubectl get nodes --show-labels) node-labels-backup.txt
```

### 3. Audit DaemonSet logic:
```sh
kubectl get daemonset node-labeler -o yaml | yq '.spec.template.spec.containers[0].command'
# Showed: ["/bin/sh", "-c", "kubectl label node $NODE zone=us-east-1a --overwrite"]
```

### 4. Check controller logs:
```sh
kubectl logs -l app=node-labeler --tail=50 | grep -i "labeling"
```

---

## Root Cause  
**Destructive label operations**:  
1. `--overwrite` flag removed all non-specified labels  
2. No change validation before application  
3. Missing protection for business-critical labels  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# 1. Rollback DaemonSet
kubectl rollout undo daemonset/node-labeler

# 2. Restore critical labels
kubectl label nodes --all gpu=true --selector='node-role/gpu=true'
kubectl label nodes --all storage=ssd --selector='beta.kubernetes.io/storage=ssd'
```

### Permanent Solution:
```yaml
# Updated DaemonSet strategy
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-labeler
spec:
  template:
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - |
          CURRENT_LABELS=$(kubectl get node $NODE -o json | jq -c '.metadata.labels')
          kubectl patch node $NODE -p "{\"metadata\":{\"labels\":{\"zone\":\"us-east-1a\",$CURRENT_LABELS}}}"
```

---

## Lessons Learned  
⚠️ **Labels are live configuration**: Overwrites immediately affect scheduling  
⚠️ **DaemonSets are powerful**: Can modify cluster state at scale  
⚠️ **Not all labels are equal**: Some are critical for operations  

---

## Prevention Framework  

### 1. Safe Labeling Practices
```sh
# Merge (not overwrite) labels
kubectl annotate node $NODE zone=us-east-1a  # Annotations for non-critical data
kubectl patch node $NODE -p '{"metadata":{"labels":{"zone":"us-east-1a"}}}'  # Strategic merge
```

### 2. Protection Policies
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sProtectedLabels
metadata:
  name: protected-node-labels
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Node"]
  parameters:
    protectedLabels: ["gpu", "storage", "topology.kubernetes.io/*"]
```

### 3. Change Control
```yaml
# ArgoCD Sync Policy
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    automated:
      prune: false  # Prevent automatic deletions
      selfHeal: false  # Require manual intervention
```

### 4. Monitoring
```yaml
# Prometheus alerts
- alert: CriticalLabelMissing
  expr: count(kube_node_labels{label_gpu!="true"}) by (node) > 0
  for: 5m
  labels:
    severity: critical
```

---

**Key Labels to Protect**:  
- Hardware: `gpu`, `fpga`, `storage`  
- Topology: `zone`, `region`, `rack`  
- Capacity: `memory`, `cpu-type`  

**Recovery Tools**:  
```sh
# Backup node labels
kubectl get nodes -o json | jq -r '.items[].metadata.labels' > node-labels-backup.json

# Diff current vs desired
kubectl diff -f restored-labels.yaml
```

**Label Management Policy**:  
```markdown
1. **Never use `--overwrite`** for node labels  
2. **Annotate** instead of label for non-scheduling data  
3. **Test changes** on a single node first  
4. **Version control** all label operations  
```

---
---

# 📘 Scenario #21: Cluster Autoscaler Thrashing from Flaky Readiness Probes

**Category**: Cluster Scaling  
**Environment**: Kubernetes v1.24, AWS EKS with Cluster Autoscaler v1.22.2  
**Impact**: 40% cost overrun from node churn, workload instability  

---

## Scenario Summary  
A deployment with an unreliable readiness probe triggered continuous node scaling cycles (5-7 per hour), causing performance degradation and cloud cost spikes.

---

## What Happened  
- **Problematic deployment**:  
  - Readiness probe checked an endpoint with 30% failure rate  
  - `kubectl get pods` showed pods alternating `Ready/NotReady`  
- **Autoscaler reaction**:  
  - Scaled up when >3 pods were `NotReady` (considered unschedulable)  
  - Scaled down 10 minutes after pods recovered  
- **Observed symptoms**:  
  - AWS EC2 `InstanceLaunchRate` exceeded account limits  
  - Node `Ready` duration histogram showed 80% <15 minutes  
  - CloudWatch showed `CPUUtilization` sawtooth pattern  

---

## Diagnosis Steps  

### 1. Identify scaling loops:
```sh
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=100 | \
  grep -E "ScaleUp|ScaleDown"
# Output showed 6 scale-up/scale-down cycles in 2 hours
```

### 2. Locate problematic pods:
```sh
kubectl get events --sort-by=.lastTimestamp | \
  grep -i "probe failed"
# Revealed deployment/frontend pods failing probes
```

### 3. Analyze probe configuration:
```sh
kubectl get deploy frontend -o yaml | yq '.spec.template.spec.containers[].readinessProbe'
# Showed HTTP probe to /health with 1s timeout
```

### 4. Verify scaling triggers:
```sh
kubectl -n kube-system describe configmap cluster-autoscaler-status | \
  grep -A5 "ScaleUp"
```

---

## Root Cause  
**Probe misalignment**:  
1. Readiness endpoint had 300ms latency spikes (exceeding 1s timeout)  
2. No retries configured (`failureThreshold: 1`)  
3. Autoscaler reacted faster than pod stabilization  

---

## Fix/Workaround  

### Immediate Stabilization:
```sh
# 1. Adjust autoscaler cooldowns
kubectl -n kube-system edit deploy cluster-autoscaler
# Add:
# --scale-down-delay-after-add=20m
# --scale-down-unneeded-time=20m

# 2. Patch problematic deployment
kubectl patch deploy frontend -p '{
  "spec":{
    "template":{
      "spec":{
        "containers":[{
          "name":"app",
          "readinessProbe":{
            "timeoutSeconds":3,
            "periodSeconds":5,
            "failureThreshold":3
          }
        }]
      }
    }
  }
}'
```

### Long-term Solution:
```yaml
# Autoscaler Helm values
autoDiscovery:
  clusterName: my-cluster
extraArgs:
  balance-similar-node-groups: true
  skip-nodes-with-system-pods: false
  scale-down-delay-after-add: 15m
  scale-down-unneeded-time: 15m
  max-node-provision-time: 10m
```

---

## Lessons Learned  
⚠️ **Probes are scaling triggers**: Flakiness causes infrastructure ripple effects  
⚠️ **Cooldowns matter**: Autoscaler defaults may be too aggressive  
⚠️ **Monitoring gaps**: Need visibility into probe failures vs scaling  

---

## Prevention Framework  

### 1. Probe Validation Checklist
```markdown
1. [ ] Test under peak load (latency >3x normal)  
2. [ ] Set `timeoutSeconds` > P99 endpoint latency  
3. [ ] Configure `failureThreshold` >=3  
4. [ ] Verify with `kubectl get --raw="/readyz?verbose"`  
```

### 2. Autoscaler Safeguards
```yaml
# Prometheus alerts
- alert: RapidNodeChurn
  expr: increase(cluster_autoscaler_scale_ups_total[1h]) > 5
  labels:
    severity: critical
  annotations:
    summary: "Cluster scaling thrashing ({{ $value }} scale-ups/hour)"

- alert: FailedProbesHigh
  expr: sum(rate(kubelet_prober_probe_total{result!="success"}[5m])) by (pod, probe_type) > 5
  labels:
    severity: warning
```

### 3. Deployment Policies
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sValidReadinessProbes
metadata:
  name: readiness-probe-timeout-check
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
  parameters:
    minTimeoutSeconds: 2
    minFailureThreshold: 3
```

### 4. Load Testing
```sh
# Simulate traffic spikes
kubectl run -it --rm vegeta --image=peterevans/vegeta -- \
  sh -c "echo 'GET http://frontend:8080/health' | vegeta attack -rate=100/s -duration=1m | vegeta report"
```

---

**Key Metrics to Monitor**:  
- `kubelet_prober_probe_total{result="failure"}`  
- `cluster_autoscaler_nodes_count`  
- `aws_ec2_running_instances` by AutoScalingGroup  

**Debugging Tools**:  
```sh
# Watch scaling decisions
kubectl -n kube-system logs -l app=cluster-autoscaler -f --tail=100 | grep "Event"

# Inspect probe history
kubectl get --raw "/api/v1/namespaces/default/pods/frontend-xyz123/proxy/debug/pprof/health?debug=2"
```
**Autoscaler Tuning Guide**:  

| Parameter                     | Production Recommendation |
|-------------------------------|--------------------------|
| scale-down-delay-after-add    | 15-30m                   |
| scale-down-unneeded-time      | 15-30m                   |
| max-node-provision-time       | 10-15m                   |
| unremovable-node-recheck-time | 5m                       |

---
---
# 📘 Scenario #22: Stale Finalizers Preventing Namespace Deletion

**Category**: Resource Lifecycle Management  
**Environment**: Kubernetes v1.21, Self-managed with Custom Operators  
**Impact**: Namespace stuck in Terminating for 72+ hours, blocking resource cleanup  

---

## Scenario Summary  
A namespace became permanently stuck in `Terminating` state due to orphaned finalizers from an uninstalled custom controller, requiring manual intervention to resolve.

---

## What Happened  
- **Controller uninstallation**:  
  - Team deleted a CustomResourceDefinition (CRD) without first cleaning up instances  
  - Operator deployment was removed while finalizers were still pending  
- **Termination deadlock**:  
  - `kubectl delete ns` command hung indefinitely  
  - Namespace status showed `phase: Terminating` for 3 days  
  - Blocked CI/CD pipelines needing namespace reuse  
- **Root discovery**:  
  - Finalizer `custom-controller/cleanup` referenced a non-existent controller  
  - API server continuously retried finalizer execution  

---

## Diagnosis Steps  

### 1. Verify namespace status:
```sh
kubectl get ns <namespace> -o jsonpath='{.status.phase}'
# Output: Terminating
```

### 2. Inspect finalizers:
```sh
kubectl get ns <namespace> -o json | jq '.spec.finalizers'
# Showed ["kubernetes", "custom-controller/cleanup"]
```

### 3. Check for controller:
```sh
kubectl get deploy -n operator-system custom-controller
# Error: Error from server (NotFound)
```

### 4. Audit CRD history:
```sh
kubectl get crd -A --show-labels | grep custom-resource
# No output (CRD deleted)
```

---

## Root Cause  
**Lifecycle violation**:  
1. Finalizer deadlock from missing controller  
2. No pre-deletion hook to clean CR instances  
3. Namespace finalizer garbage collection blocked  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Force remove finalizers
kubectl patch ns <namespace> -p '{"spec":{"finalizers":[]}}' --type=merge

# 2. Verify deletion completes
kubectl get ns <namespace> --watch
```

### Proper Cleanup Procedure:
```sh
# For future uninstalls:
kubectl delete all --all -n <namespace>  # Delete standard resources
kubectl delete <crd-name> --all -n <namespace>  # Delete CR instances 
kubectl delete crd <crd-name>  # Remove CRD last
```

---

## Lessons Learned  
⚠️ **Finalizers are promises**: Must have operational controllers  
⚠️ **Order matters**: Always delete resources before their definitions  
⚠️ **Namespace termination is atomic**: Single finalizer blocks everything  

---

## Prevention Framework  

### 1. Uninstall Automation
```yaml
# Helm pre-delete hook
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    "helm.sh/hook": pre-delete
spec:
  template:
    spec:
      containers:
      - name: cleanup
        command: ["/bin/sh", "-c", "kubectl patch $(kubectl get <crd> -o name) -p '{\"metadata\":{\"finalizers\":[]}}' --type=merge"]
```

### 2. Finalizer Auditing
```sh
# Daily finalizer health check
kubectl get ns -o json | jq -r '.items[] | select(.status.phase=="Terminating") | .metadata.name'
kubectl get crd -o json | jq -r '.items[] | select(.spec.scope=="Namespaced") | .metadata.name' | xargs -I{} sh -c "kubectl get {} -A -o json | jq -r '.items[] | select(.metadata.finalizers!=null) | .metadata.name'"
```

### 3. Protection Policies
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: CRDHasFinalizerController
metadata:
  name: crd-finalizer-controller-check
spec:
  match:
    kinds:
    - apiGroups: ["apiextensions.k8s.io"]
      kinds: ["CustomResourceDefinition"]
  parameters:
    requiredAnnotations:
      finalizerController: "true"
```

### 4. Monitoring
```yaml
# Prometheus alerts
- alert: StaleFinalizers
  expr: time() - kube_namespace_status_phase{phase="Terminating"} > 3600
  labels:
    severity: warning
  annotations:
    description: Namespace {{ $labels.namespace }} stuck terminating for 1h
```

---

**Key Resources to Check**:  
- Namespaces in `Terminating` >15m  
- Custom Resources with finalizers  
- Orphaned finalizers in etcd  

**Debugging Tools**:  
```sh
# List all resources with finalizers
kubectl api-resources --verbs=list -o name | xargs -n1 kubectl get -A --ignore-not-found -o json | jq -r '.items[] | select(.metadata.finalizers!=null) | "\(.kind)/\(.metadata.name)"'

# Check etcd for stale keys
etcdctl get / --prefix --keys-only | grep finalizers
```

**Finalizer Best Practices**:  
```markdown
1. **Implement timeout logic** - Finalizers should self-clean after deadline  
2. **Use owner references** - Ensure child resources don't block parents  
3. **Document all finalizers** - Maintain registry with ownership  
4. **Test uninstall scenarios** - Include in CI/CD pipelines  
```
---
---
# 📘 Scenario #23: CoreDNS CrashLoop Due to Invalid ConfigMap Update

**Category**: Cluster Networking  
**Environment**: Kubernetes 1.23, GKE (Managed Control Plane)  
**Impact**: Cluster-wide DNS outage lasting 47 minutes  

---

## Scenario Summary  
A malformed CoreDNS ConfigMap update caused all DNS resolution to fail, breaking service discovery and pod-to-pod communication across the entire cluster.

---

## What Happened  
- **Config change**:  
  - Added rewrite rule with incorrect syntax: `rewrite stop name regex (.*)\.internal internal.svc.cluster.local`  
  - Missing plugin declaration in Corefile preamble  
- **Immediate impact**:  
  - CoreDNS pods entered `CrashLoopBackOff`  
  - `kubectl logs` showed `Corefile:5 - Error during parsing: Unknown directive 'rewrit'`  
- **Cascading failures**:  
  - Service mesh (Istio) sidecars failed health checks  
  - `kubelet` reported `node not ready` due to DNS timeouts  

---

## Diagnosis Steps  

### 1. Verify DNS availability:
```sh
kubectl run -it --rm dns-test --image=busybox -- nslookup kubernetes.default
# Error: `Server failure`
```

### 2. Check CoreDNS status:
```sh
kubectl -n kube-system get pods -l k8s-app=kube-dns
# Showed CrashLoopBackOff
```

### 3. Inspect pod logs:
```sh
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
# Output: `plugin/rewrite: unknown property "stop"`
```

### 4. Validate ConfigMap:
```sh
kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}' | grep -A5 rewrite
# Revealed malformed rewrite rule
```

---

## Root Cause  
**Configuration error**:  
1. Missing `rewrite` plugin in CoreDNS deployment  
2. Typo in rewrite directive (`rewrit` vs `rewrite`)  
3. No validation before applying changes  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Rollback ConfigMap
kubectl -n kube-system rollout undo configmap/coredns

# 2. Force pod restart
kubectl -n kube-system rollout restart deployment/coredns

# 3. Verify restoration
kubectl run -it --rm dns-test --image=busybox -- nslookup kubernetes.default
```

### Correct Configuration:
```corefile
# Valid rewrite rule
.:53 {
    rewrite name regex (.*)\.internal internal.svc.cluster.local
    # ... other plugins ...
}
```

---

## Lessons Learned  
⚠️ **DNS is critical infrastructure**: Breaks within seconds of bad config  
⚠️ **CoreDNS is unforgiving**: Silent until reload, then hard fails  
⚠️ **Managed ≠ foolproof**: GKE doesn't validate Corefile syntax  

---

## Prevention Framework  

### 1. Validation Workflow
```sh
# Pre-apply check using official image
docker run -i coredns/coredns:1.8.6 -conf - <<< "$(kubectl get configmap/coredns -n kube-system -o jsonpath='{.data.Corefile}')"
```

### 2. GitOps Safeguards
```yaml
# ArgoCD Application with pre-sync hook
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - Validate=true
    hooks:
      preSync:
        - name: validate-coredns
          template: coredns-validator
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: CoreDNSDown
  expr: absent(up{job="kube-dns"} == 1)
  for: 1m
  labels:
    severity: critical
```

### 4. Change Management
```markdown
## CoreDNS Config Checklist
1. [ ] Test in staging cluster  
2. [ ] Validate with `coredns -conf`  
3. [ ] Document change in runbook  
4. [ ] Prepare rollback procedure  
```

---

**Key Metrics to Monitor**:  
- `coredns_dns_responses_total{rcode="SERVFAIL"}`  
- `kubelet_dns_errors`  
- `probe_dns_lookup_time_seconds`  

**Debugging Tools**:  
```sh
# Live DNS query test
kubectl run -it --rm dns-debug --image=nicolaka/netshoot -- dig +trace kubernetes.default.svc.cluster.local

# CoreDNS health check
kubectl get --raw "/api/v1/namespaces/kube-system/services/coredns:9153/proxy/health"
```

**CoreDNS Best Practices**:  
```corefile
# Minimal production config
.:53 {
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
# 📘 Scenario #24: Pod Eviction Storm Due to DiskPressure

**Category**: Node Resource Management  
**Environment**: Kubernetes 1.25, Self-managed, containerd runtime  
**Impact**: 83% of cluster pods evicted, 2-hour service disruption  

---

## Scenario Summary  
A mass container image update triggered uncontrolled disk consumption across all worker nodes, causing cascading pod evictions and cluster instability.

---

## What Happened  
- **Batch update initiated**:  
  - Deployment rollout of 5GB image to 1,200 pods  
  - Simultaneous pulls across 50 nodes  
- **Storage exhaustion**:  
  - `/var/lib/containerd` reached 100% utilization in 8 minutes  
  - `kubelet` enforced `DiskPressure` evictions (`nodefs.available<10%`)  
- **Cascading failures**:  
  - System pods (CNI, CSI) evicted first  
  - Cluster-autoscaler spun up new nodes (which also filled)  
  - etcd overwhelmed by pod churn events  

---

## Diagnosis Steps  

### 1. Verify node conditions:
```sh
kubectl get nodes -o json | jq -r '.items[] | .metadata.name + " " + (.status.conditions[] | select(.type=="DiskPressure") | .status'
```

### 2. Check disk usage:
```sh
kubectl debug node/<node> -it --image=alpine -- df -h /var/lib/containerd
# Output: 100% used
```

### 3. Inspect image cache:
```sh
kubectl debug node/<node> -it --image=ubuntu -- crictl images --digests
# Showed 40+ GB of images
```

### 4. Analyze eviction logs:
```sh
journalctl -u kubelet --no-pager -n 100 | grep -A10 "DiskPressure"
# Logged "Attempting to reclaim ephemeral-storage"
```

---

## Root Cause  
**Unmanaged disk consumption**:  
1. No image garbage collection thresholds  
2. Unlimited concurrent image pulls  
3. Default 100GB node disks insufficient for workload churn  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Pause deployments
kubectl rollout pause deploy/<batch-job>

# 2. Manual image pruning (on affected nodes)
crictl rmi --prune

# 3. Taint nodes to prevent rescheduling
kubectl taint nodes <node> disk-pressure=cleanup:NoSchedule
```

### Long-term Solution:
```yaml
# Kubelet configuration
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  nodefs.available: "15%"
  imagefs.available: "20%"
evictionMinimumReclaim:
  nodefs.available: "5Gi"
  imagefs.available: "3Gi"
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
```

---

## Lessons Learned  
⚠️ **DiskPressure is non-forgiving**: Evictions start before OOM kills  
⚠️ **Storage is a shared resource**: One pod can starve the node  
⚠️ **Default thresholds are dangerous**: Must be tuned per workload  

---

## Prevention Framework  

### 1. Resource Management
```yaml
# Pod storage limits
resources:
  limits:
    ephemeral-storage: "2Gi"
  requests:
    ephemeral-storage: "1Gi"
```

### 2. Deployment Safeguards
```yaml
# RollingUpdate strategy
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 10%
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: NodeDiskPressureImminent
  expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.2
  for: 5m
  labels:
    severity: critical
```

### 4. Node Design
```sh
# Separate 200GB volume for /var/lib/containerd
mkfs.xfs /dev/nvme1n1
mkdir -p /var/lib/containerd
mount /dev/nvme1n1 /var/lib/containerd
```

---

**Key Metrics to Monitor**:  
- `container_fs_usage_bytes`  
- `kubelet_evictions` by `DiskPressure`  
- `crictl_images_bytes`  

**Debugging Tools**:  
```sh
# Live disk usage
kubectl debug node/<node> -it --image=nicolaka/netshoot -- bmon

# Find large images
kubectl debug node/<node> -it --image=ubuntu -- \
  du -ah /var/lib/containerd | sort -rh | head -20
```

**Image GC Policy**:  
```markdown
| Parameter                     | Production Value |
|-------------------------------|------------------|
| imageGCHighThresholdPercent   | 85               |
| imageGCLowThresholdPercent    | 75               |
| imageMinimumGCAge            | 2h               |
```

---
---
# 📘 Scenario #25: Orphaned PVs Causing Unscheduled Pods

**Category**: Storage Management  
**Environment**: Kubernetes 1.20, vSphere CSI Driver 2.3  
**Impact**: 47 PVCs stuck in Pending state, blocking application deployments  

---

## Scenario Summary  
PersistentVolumes (PVs) left in `Released` state after pod deletions prevented new PVCs from binding, causing critical workloads to fail scheduling.

---

## What Happened  
- **Storage lifecycle gap**:  
  - PVs created with `persistentVolumeReclaimPolicy: Retain`  
  - No cleanup process for released volumes  
- **Provisioning failure**:  
  - New PVCs for same storage class failed with `waiting for first consumer to be created`  
  - vSphere CSI logs showed `volume already exists` errors  
- **Cascading effects**:  
  - StatefulSet pods stuck in `Pending`  
  - Database initialization jobs timed out  

---

## Diagnosis Steps  

### 1. Verify PVC status:
```sh
kubectl get pvc -A -o wide | grep -v Bound
# Showed multiple PVCs pending
```

### 2. Check PV inventory:
```sh
kubectl get pv --sort-by=.metadata.creationTimestamp \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.claimRef.name,RECLAIM:.spec.persistentVolumeReclaimPolicy
# Revealed 20+ PVs in Released state
```

### 3. Inspect storage provider:
```sh
kubectl logs -n kube-system ds/vsphere-csi-node -c vsphere-csi-driver | grep -A10 "already exists"
```

### 4. Audit reclaim policies:
```sh
kubectl get sc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.reclaimPolicy}{"\n"}{end}'
# Showed 80% volumes configured with Retain
```

---

## Root Cause  
**Storage lifecycle breakdown**:  
1. `Retain` policy required manual intervention  
2. CSI driver couldn't reprovision existing volumes  
3. No monitoring for orphaned PVs  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Delete orphaned PVs (after data backup)
kubectl get pv | grep Released | awk '{print $1}' | xargs -I{} kubectl delete pv {}

# 2. Retry pending PVCs
kubectl get pvc -A | grep Pending | awk '{print $1,$2}' | \
  xargs -n2 sh -c 'kubectl patch pvc -n $0 $1 -p '"'"'{"metadata":{"annotations":{"volume.beta.kubernetes.io/storage-class":"force-retry"}}'"'"'
```

### Long-term Solution:
```yaml
# StorageClass update
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vsphere-ssd
provisioner: csi.vsphere.vmware.com
reclaimPolicy: Delete  # Changed from Retain
volumeBindingMode: WaitForFirstConsumer
```

---

## Lessons Learned  
⚠️ **Released ≠ Recycled**: PVs require explicit cleanup  
⚠️ **Retain policies are dangerous**: Need automated follow-up  
⚠️ **Storage is stateful**: Must track across entire lifecycle  

---

## Prevention Framework  

### 1. Automated Cleanup
```yaml
# K8s Job to clean Released PVs (runs weekly)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pv-cleaner
spec:
  schedule: "0 3 * * 6"  # Saturday 3AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleaner
            image: bitnami/kubectl
            command:
            - /bin/sh
            - -c
            - |
              kubectl get pv --no-headers | \
                grep Released | \
                awk '{print $1}' | \
                xargs kubectl delete pv
```

### 2. Storage Policy
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sValidStorageClass
metadata:
  name: require-delete-policy
spec:
  match:
    kinds:
    - apiGroups: ["storage.k8s.io"]
      kinds: ["StorageClass"]
  parameters:
    allowedReclaimPolicies: ["Delete"]
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: OrphanedPVs
  expr: count(kube_persistentvolume_status_phase{phase="Released"}) > 0
  for: 1h
  labels:
    severity: warning

- alert: PendingPVCs
  expr: count(kube_persistentvolumeclaim_status_phase{phase="Pending"}) > 5
  for: 30m
  labels:
    severity: critical
```

### 4. Documentation

## PV Cleanup Protocol
1. **Identify candidates**:
   ```sh
   kubectl get pv -o json | jq -r '.items[] | select(.status.phase=="Released") | .metadata.name'
   ```
2. **Backup data** (if needed):
   ```sh
   velero backup create pv-cleanup-$(date +%s) --include-resources persistentvolumes
   ```
3. **Release volumes**:
   ```sh
   kubectl delete pv <name> --wait=false
   ```

---

**Key Metrics to Monitor**:  
- `kube_persistentvolume_status_phase`  
- `kube_persistentvolumeclaim_status_phase`  
- `csi_volume_operations_total`  

**Debugging Tools**:  
```sh
# Find PVCs waiting for PVs
kubectl get pvc -A -o json | jq -r '.items[] | select(.status.phase=="Pending") | .metadata.namespace + "/" + .metadata.name'

# Check storage provider capacity
kubectl get --raw="/apis/storage.k8s.io/v1/csinodes" | jq '.items[].spec.drivers[]'
```

**PV Lifecycle Best Practices**:  
```markdown
1. **Default to Delete policy** unless data retention required  
2. **Tag resources** with owner/expiration metadata  
3. **Implement backup** before cleanup  
4. **Monitor volume age** - alert if >30 days unused  
```
---
---
# 📘 Scenario #26: Taints and Tolerations Mismatch Prevented Workload Scheduling

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.22, AKS GPU Node Pool  
**Impact**: GPU-accelerated workloads failed to schedule for 6+ hours  

---

## Scenario Summary  
New GPU nodes remained underutilized because critical workloads lacked required tolerations, causing scheduling deadlocks and resource starvation for AI/ML pipelines.

---

## What Happened  
- **Infrastructure upgrade**:  
  - New `NCv3` node pool added with `node-role.kubernetes.io/gpu:NoSchedule`  
  - Node autoscaler provisioned 8 GPU nodes ($12/hr each)  
- **Scheduling failures**:  
  - `kubectl get events` showed `0/8 nodes available: 8 node(s) had untolerated taint`  
  - GPU utilization metrics showed 0% usage  
- **Diagnosis findings**:  
  - 47 deployments lacked GPU tolerations  
  - Node selector `accelerator: nvidia-tesla` still pointed to old nodes  

---

## Diagnosis Steps  

### 1. Check pending pods:
```sh
kubectl get pods -A --field-selector status.phase=Pending -o wide | grep gpu
```

### 2. Inspect scheduling barriers:
```sh
kubectl describe pod <pending-pod> | grep -A10 Events
# Showed "node(s) had untolerated taint {node-role.kubernetes.io/gpu: }"
```

### 3. Verify node taints:
```sh
kubectl get nodes -l accelerator=nvidia-tesla -o json | \
  jq -r '.items[].spec.taints'
# Output: [{"effect":"NoSchedule","key":"node-role.kubernetes.io/gpu"}]
```

### 4. Audit workload specs:
```sh
kubectl get deploy -A -o json | \
  jq -r '.items[] | select(.spec.template.spec.nodeSelector?.accelerator=="nvidia-tesla") | 
  .metadata.namespace + "/" + .metadata.name' | \
  xargs -I{} kubectl get deploy -n {} -o json | \
  jq -r '.spec.template.spec.tolerations' | grep -L "node-role.kubernetes.io/gpu"
```

---

## Root Cause  
**Scheduling policy drift**:  
1. New taints introduced without workload updates  
2. CI/CD pipelines didn't enforce toleration standards  
3. No validation between node selectors and taints  

---

## Fix/Workaround  

### Immediate Resolution:
```yaml
# Patch deployments with tolerations
kubectl patch deploy <name> -p '{
  "spec":{
    "template":{
      "spec":{
        "tolerations":[{
          "key":"node-role.kubernetes.io/gpu",
          "operator":"Exists",
          "effect":"NoSchedule"
        }]
      }
    }
  }
}'
```

### Long-term Solution:
```yaml
# Kustomize patch for GPU workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-workload
spec:
  template:
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
```

---

## Lessons Learned  
⚠️ **Taints break scheduling silently**: No warnings during apply  
⚠️ **Node selectors ≠ taints**: Both must be coordinated  
⚠️ **Costly idle resources**: Untolerated GPU nodes waste $100+/hr  

---

## Prevention Framework  

### 1. Admission Control
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredTolerations
metadata:
  name: gpu-toleration-check
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment","DaemonSet"]
    labelSelector:
      matchExpressions:
      - key: accelerator
        operator: In
        values: ["nvidia-tesla"]
  parameters:
    tolerations:
    - key: "node-role.kubernetes.io/gpu"
      operator: "Exists"
```

### 2. CI/CD Validation
```sh
# Pre-deployment check
if grep -q "nvidia-tesla" $MANIFEST && ! grep -q "node-role.kubernetes.io/gpu" $MANIFEST; then
  echo "ERROR: GPU workloads require tolerations"
  exit 1
fi
```

### 3. Node Pool Testing
```yaml
# Test deployment template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-test
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        accelerator: nvidia-tesla
      tolerations:
      - key: "node-role.kubernetes.io/gpu"
        operator: "Exists"
      containers:
      - name: stress
        image: nvidia/cuda:11.0-base
        command: ["nvidia-smi"]
```

### 4. Monitoring
```yaml
# Prometheus alerts
- alert: UntoleratedTaints
  expr: count(kube_pod_info{node=~".*gpu.*"} unless on(pod) kube_pod_spec_tolerations{key="node-role.kubernetes.io/gpu"} > 0
  for: 15m
  labels:
    severity: critical
```

---

**Key Metrics to Monitor**:  
- `kube_node_spec_taints`  
- `kube_pod_spec_tolerations`  
- `nvidia_gpu_duty_cycle` 

**Debugging Tools**:  
```sh
# List nodes with taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check toleration coverage
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.nodeSelector?.accelerator=="nvidia-tesla") | .metadata.name + " " + (.spec.tolerations // [] | map(.key) | join(","))'
```

**Toleration Best Practices**:  
```markdown
1. **Standardize tolerations** per node pool type  
2. **Use `Exists` operator** for role-based taints  
3. **Document taints** in cluster runbooks  
4. **Test before scaling** new node pools  
```
---
---

# 📘 Scenario #27: Node Bootstrap Failure Due to Unavailable Container Registry

**Category**: Cluster Provisioning  
**Environment**: Kubernetes 1.21, On-prem, Air-gapped Registry  
**Impact**: 100% node provisioning failure during registry outage  

---

## Scenario Summary  
A private container registry outage prevented new nodes from joining the cluster by blocking pulls of critical bootstrap images (`pause`, `kube-proxy`, CNI), causing complete failure of scaling operations.

---

## What Happened  
- **Registry maintenance**:  
  - Storage backend upgrade caused 2-hour registry unavailability  
  - No maintenance window coordination with cluster operations  
- **Bootstrap failures**:  
  - `containerd` logged `failed to pull image: registry.internal:5000/pause:3.4.1`  
  - Nodes stuck in `NotReady` with `ContainerRuntimeNotReady` condition  
- **Cascading effects**:  
  - Cluster autoscaler triggered 15 failed node launches  
  - Emergency manual scaling attempts also failed  

---

## Diagnosis Steps  

### 1. Check node readiness:
```sh
kubectl get nodes -o json | \
  jq -r '.items[] | .metadata.name + " " + (.status.conditions[] | select(.type=="Ready") | .status + " " + .message'
```

### 2. Inspect container runtime:
```sh
journalctl -u containerd --no-pager -n 100 | grep -i pull
# Output: "PullImage registry.internal:5000/pause:3.4.1 failed"
```

### 3. Verify registry access:
```sh
curl -I https://registry.internal:5000/v2/
# Returned 503 Service Unavailable
```

### 4. Check essential images:
```sh
kubeadm config images list --kubernetes-version v1.21.5
# All images pointed to registry.internal
```

---

## Root Cause  
**Hard dependency on registry**:  
1. No local image cache for critical components  
2. No fallback mirror registry configured  
3. Infrastructure-as-Code (IaC) templates lacked image preloading  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# 1. Restore registry service
systemctl restart registry-backend

# 2. Manually load images on affected nodes
ctr -n k8s.io images import /opt/k8s/images/pause.tar
```


### Long-term Solution:
```yaml
# Packer template snippet for node AMI
provisioner "shell" {
  script = "preload-images.sh"
  environment_vars = [
    "KUBE_VERSION=v1.21.5",
    "REGISTRY=registry.internal:5000"
  ]
}
```

---

## Lessons Learned  
⚠️ **Bootstrap is fragile**: Requires all dependencies available  
⚠️ **Air-gapped needs redundancy**: Must have backup image sources  
⚠️ **Node images should be atomic**: Include all needed containers  

---

## Prevention Framework  

### 1. Image Preloading
```sh
# preload-images.sh
IMAGES=$(kubeadm config images list --kubernetes-version $KUBE_VERSION)
for image in $IMAGES; do
  docker pull $REGISTRY/${image#*/}
  docker save -o /opt/k8s/images/${image##*/}.tar $REGISTRY/${image#*/}
done
```

### 2. Registry Redundancy
```yaml
# containerd config.toml
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.internal"]
      endpoint = ["https://registry-01.internal", "https://registry-02.internal"]
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: RegistryUnavailable
  expr: probe_success{job="registry-healthcheck"} == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Container registry {{ $labels.instance }} is down"
```

### 4. Provisioning Checks
```yaml
# Ansible pre-task
- name: Verify registry access
  uri:
    url: "https://{{ registry_url }}/v2/"
    status_code: 200
    timeout: 5
  register: registry_check
  failed_when: false
  changed_when: false

- name: Fail if registry unreachable
  fail:
    msg: "Registry {{ registry_url }} is unavailable"
  when: registry_check.status != 200
```

---

**Key Images to Preload**:  
- `pause` (kubelet sandbox)  
- CNI plugins (`calico`, `flannel`)  
- `kube-proxy`  
- `node-exporter`  

**Debugging Tools**:  
```sh
# Check loaded images
ctr -n k8s.io images list

# Verify image pull secrets
kubectl get secret -n kube-system | grep registry

# Test registry access
kubectl run -it --rm registry-test --image=alpine -- \
  sh -c "apk add curl && curl -u user:pass https://registry.internal:5000/v2/_catalog"
```
