# 📘 Scenario 32: Inconsistent Node Labels Causing Scheduling Bugs

**Category**: Cluster Scheduling  
**Environment**: Kubernetes 1.24, Multi-zone GKE  
**Impact**: Critical workloads failed to schedule, breaking zone-balancing guarantees  

---

## Scenario Summary  
Missing `topology.kubernetes.io/zone` labels on manually provisioned nodes caused topology-aware workloads to fail scheduling, violating high-availability requirements.

---

## What Happened  
- **Manual node provisioning**:  
  - Emergency capacity added via `gcloud compute instances create`  
  - Missing cloud-controller-manager auto-labeling  
- **Scheduling failures**:  
  - Pods with `topologySpreadConstraints` stuck in `Pending`  
  - `kubectl describe pod` showed `0/3 nodes are available: 3 node(s) missing required label`  
- **HA impact**:  
  - 5-node cluster had only 2 labeled zones instead of 3  
  - StatefulSet pods concentrated in single zone  

---

## Diagnosis Steps  

### 1. Identify pending pods:
```sh
kubectl get pods -A --field-selector status.phase=Pending -o wide
```

### 2. Check scheduling events:
```sh
kubectl describe pod <pending-pod> | grep -A10 "Events"
# Output: "0/3 nodes are available: 3 node(s) missing required label topology.kubernetes.io/zone"
```


### 3. Audit node labels:
```sh
kubectl get nodes -o json | jq -r '.items[] | .metadata.name + " " + (.metadata.labels | to_entries | map(select(.key | contains("zone"))) | map("\(.key)=\(.value)") | join(","))'
# Showed 3 nodes without zone labels
```

### 4. Verify topology constraints:
```sh
kubectl get pods -o json | jq -r '.items[] | select(.spec.topologySpreadConstraints) | .metadata.name'
```

---

## Root Cause  
**Label injection gap**:  
1. Manual node creation bypassed cloud provider auto-labeling  
2. No validation for required topology labels  
3. Missing fallback labeling mechanism  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Identify unlabeled nodes
kubectl get nodes --show-labels | grep -v "topology.kubernetes.io/zone"

# 2. Apply zone labels manually
kubectl label nodes <node1> <node2> topology.kubernetes.io/zone=us-central1-a

# 3. Trigger rescheduling
kubectl patch deployment <app> -p '{"spec":{"template":{"metadata":{"annotations":{"restartedAt":"'$(date +%s)'"}}}}}'
```

### Long-term Solution:
```yaml
# DaemonSet for label enforcement
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-labeler
spec:
  template:
    spec:
      containers:
      - name: labeler
        image: bitnami/kubectl
        command:
        - /bin/sh
        - -c
        - |
          ZONE=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone | cut -d/ -f4)
          kubectl label node $NODE topology.kubernetes.io/zone=$ZONE --overwrite
```

---

## Lessons Learned  
⚠️ **Manual operations break automation**: Cloud labels require provider integration  
⚠️ **Topology is foundational**: Missing labels break critical scheduling features  
⚠️ **Silent degradation**: Workloads fail scheduling without clear root cause  

---

## Prevention Framework  

### 1. Automated Labeling
```yaml
# Cloud-init script for manual nodes
#cloud-config
runcmd:
  - |
    ZONE=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone | cut -d/ -f4)
    kubectl label node $(hostname) topology.kubernetes.io/zone=$ZONE
```

### 2. Admission Control
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredNodeLabels
metadata:
  name: require-topology-labels
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Node"]
  parameters:
    labels:
    - key: topology.kubernetes.io/zone
    - key: topology.kubernetes.io/region
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: MissingTopologyLabels
  expr: count(kube_node_labels{label_topology_kubernetes_io_zone=""}) > 0
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Nodes missing topology labels ({{ $value }})"
```

### 4. CI/CD Validation
```sh
# Pre-join node validation
if ! kubectl auth can-i create nodes; then
  echo "ERROR: Use automated node pools instead of manual creation"
  exit 1
fi
```

---

**Key Topology Labels to Enforce**:  
- `topology.kubernetes.io/zone`  
- `topology.kubernetes.io/region`  
- `kubernetes.io/hostname`  

**Debugging Tools**:  
```sh
# Check topology distribution
kubectl get nodes -o json | jq -r '.items[] | .metadata.labels["topology.kubernetes.io/zone"]' | sort | uniq -c

# Verify spread constraints
kubectl get pods -o json | jq -r '.items[] | select(.spec.topologySpreadConstraints) | "\(.metadata.name): \(.spec.topologySpreadConstraints[].topologyKey)"'

# Test scheduling simulation
kubectl create job test-schedule --image=busybox --dry-run=client -o yaml | kubectl apply -f -
```

**Topology Spread Best Practices**:  
```markdown
1. **Always use cloud node pools** for automatic labeling  
2. **Validate labels pre-production**:  
   ```sh
   kubectl get nodes -o json | jq -r '.items[].metadata.labels | has("topology.kubernetes.io/zone")' | grep -q false && echo "MISSING LABELS"
   ```
3. **Test topology constraints** with canary deployments  
4. **Monitor zone distribution** of critical workloads  
```