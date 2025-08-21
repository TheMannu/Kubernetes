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