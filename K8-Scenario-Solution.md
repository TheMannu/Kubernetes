# 📘 Scenario #1: Zombie Pods Causing Node Drain to Hang

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


# 📘 Scenario #2: API Server Crash Due to Excessive CRD Writes  

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

# 📘 Scenario #3: Node Not Rejoining After Reboot

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