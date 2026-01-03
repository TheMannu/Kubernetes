# 📘 Scenario 47: Control Plane Overload Due to High Audit Log Volume

**Category**: Control Plane Performance & Security  
**Environment**: Kubernetes 1.22, Azure AKS, 1000+ nodes  
**Impact**: API server latency increased to 15s+, cluster operations slowed to a crawl  

---

## Scenario Summary  
An overly permissive audit logging policy generated massive volumes of audit events, overwhelming the API server and etcd, causing severe performance degradation across the entire cluster.

---

## What Happened  
- **Audit policy misconfiguration**:  
  - Policy set to log all events at `Metadata` level for all resources  
  - No filters for high-frequency operations (GET, LIST, WATCH)  
- **Performance symptoms**:  
  - API server CPU usage spiked to 95%+  
  - etcd disk I/O saturated with audit log writes  
  - `kubectl` commands timed out after 30s  
  - Audit log storage grew at 50GB/hour  
- **Root discovery**:  
  - Single namespace generated 2M audit events/hour  
  - 80% of logs were repetitive `get` and `list` operations  

---

## Diagnosis Steps  

### 1. Check API server metrics:
```sh
kubectl get --raw /metrics | grep -E "apiserver_audit_event_total|apiserver_request_duration_seconds"
# Showed 500k+ audit events per minute
```
 
### 2. Monitor API server resource usage:
```sh
kubectl top pods -n kube-system -l component=kube-apiserver
# Output: CPU 9500m/10000m, MEM 6Gi/8Gi
```

### 3. Inspect audit log volume:
```sh
# On control plane node
du -sh /var/log/kube-apiserver/audit.log*
# Output: 450GB total audit logs
```

### 4. Analyze audit policy:
```sh
kubectl -n kube-system get configmap kube-apiserver -o yaml | \
  yq '.data."audit-policy.yaml"'
# Showed overly broad rules with no request filters
```

---

## Root Cause  
**Unconstrained audit logging**:  
1. Policy logging all resource types at `Metadata` level  
2. No request filters for high-volume operations  
3. Missing audit log rotation and retention limits  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Temporarily reduce audit level (requires API server restart)
kubectl -n kube-system edit configmap kube-apiserver
# Change default audit level from Metadata to None

# 2. Restart API servers (AKS managed - requires support ticket)
# Submit Azure support ticket for emergency API server restart

# 3. Clean up audit logs
kubectl debug node/<control-plane> -it --image=alpine -- \
  find /var/log/kube-apiserver -name "audit.log*" -mtime +1 -delete
```

### Long-term Solution:
```yaml
# Refined audit policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services", "services/status"]
    
- level: None
  userGroups: ["system:nodes"]
  verbs: ["get"]
  resources:
  - group: ""
    resources: ["nodes", "nodes/status"]
        
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
  - group: "apps"
  - group: "authentication.k8s.io"
    
- level: RequestResponse
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
    
- level: Metadata
  verbs: ["get", "list", "watch"]
  
- level: None
  resources:
  - group: ""
    resources: ["events"]
```

---

## Lessons Learned  
⚠️ **Audit logs have performance costs**: Each log entry consumes CPU and I/O  
⚠️ **Filtering is essential**: Must exclude high-frequency, low-value operations  
⚠️ **Managed services have limitations**: AKS audit configuration requires planning  

---