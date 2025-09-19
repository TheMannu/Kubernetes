# 📘 Scenario 20: Node Labels Accidentally Overwritten by DaemonSet

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

