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