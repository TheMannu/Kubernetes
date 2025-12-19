# 📘 Scenario 43: Node Pool Scaling Impacting StatefulSets

**Category**: Storage & Scheduling  
**Environment**: Kubernetes 1.24, GKE with Regional Clusters  
**Impact**: StatefulSet pod disruptions causing database corruption and application downtime  

---

## Scenario Summary  
Automatic node pool scaling triggered involuntary StatefulSet pod migrations, breaking persistent volume attachments and causing data consistency issues in stateful workloads.

---

## What Happened  
- **Cluster autoscaling event**:  
  - GKE autoscaler removed 3 nodes during cost optimization cycle  
  - StatefulSet pods using `volumeBindingMode: WaitForFirstConsumer`  
  - PVCs remained bound to original nodes' zones  
- **Failure symptoms**:  
  - StatefulSet pods stuck in `Pending` with `volume node affinity conflict`  
  - Database pods (Cassandra, PostgreSQL) lost quorum  
  - `kubectl describe pvc` showed `no available persistent volumes`  
- **Data integrity risks**:  
  - Multi-node databases experienced split-brain scenarios  
  - Write-ahead logs became inconsistent  

---

## Diagnosis Steps  

### 1. Identify affected StatefulSets:
```sh
kubectl get statefulsets -A -o json | \
  jq -r '.items[] | select(.spec.volumeClaimTemplates) | .metadata.namespace + "/" + .metadata.name'
```

### 2. Check PVC binding status:
```sh
kubectl get pvc -A -o wide | grep -v Bound
# Showed multiple PVCs in Pending state
```

### 3. Inspect volume affinity errors:
```sh
kubectl describe pvc data-cassandra-0 -n database | grep -A5 Events
# Output: "no persistent volumes available for this claim due to storage class node affinity conflict"
```

### 4. Verify node topology:
```sh
kubectl get nodes -o json | jq -r '.items[] | .metadata.labels."topology.kubernetes.io/zone"'
# Showed missing zones after scaling
```

---

## Root Cause  
**Topology-aware volume constraints**:  
1. Regional StorageClass with `volumeBindingMode: WaitForFirstConsumer`  
2. No pod anti-affinity or node affinity for StatefulSet pods  
3. PVCs bound to specific zones that became unavailable  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Prevent further scaling during recovery
gcloud container clusters update my-cluster --no-enable-autoscaling

# 2. Patch StatefulSet with pod anti-affinity
kubectl patch statefulset cassandra -n database -p '{
  "spec": {
    "template": {
      "spec": {
        "affinity": {
          "podAntiAffinity": {
            "requiredDuringSchedulingIgnoredDuringExecution": [{
              "labelSelector": {
                "matchExpressions": [{
                  "key": "app",
                  "operator": "In",
                  "values": ["cassandra"]
                }]
              },
              "topologyKey": "kubernetes.io/hostname"
            }]
          }
        }
      }
    }
  }
}'

# 3. Delete and recreate PVCs (if data backed up)
kubectl delete pvc --all -n database
```

### Long-term Solution:
```yaml
# Regional StorageClass configuration
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.gke.io/zone
    values:
    - us-central1-a
    - us-central1-b
    - us-central1-c
```

---

## Lessons Learned  
⚠️ **StatefulSets are topology-sensitive**: Must consider zone/region constraints  
⚠️ **Autoscaling breaks assumptions**: Nodes are ephemeral for stateful workloads  
⚠️ **Volume binding matters**: `WaitForFirstConsumer` vs `Immediate` has major implications  

---

## Prevention Framework  

### 1. StatefulSet Design Patterns
```yaml
# StatefulSet with proper affinity and topology
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["database"]
            topologyKey: "topology.kubernetes.io/zone"
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: database
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: regional-ssd
      resources:
        requests:
          storage: 100Gi
```

### 2. Node Pool Management
```yaml
# GKE NodePool configuration for stateful workloads
apiVersion: container.googleapis.com/v1beta1
kind: NodePool
metadata:
  name: stateful-pool
spec:
  autoscaling:
    enabled: true
    minNodeCount: 3  # Minimum for quorum
    maxNodeCount: 6
  management:
    autoRepair: true
    autoUpgrade: true
  config:
    labels:
      workload-type: stateful
    taints:
    - key: workload-type
      value: stateful
      effect: NoSchedule
```

### 3. Monitoring
```yaml
# Prometheus alerts for StatefulSet health
- alert: StatefulSetPodDisruption
  expr: kube_statefulset_status_replicas_ready != kube_statefulset_status_replicas
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} has {{ $value }} unavailable pods"

- alert: PVCBindingIssues
  expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} > 0
  for: 10m
  labels:
    severity: warning
```

### 4. Scaling Safeguards
```sh
# Pre-scaling validation for stateful workloads
validate_stateful_scaling() {
  local stateful_pods=$(kubectl get pods -l workload-type=stateful -o name | wc -l)
  local available_nodes=$(kubectl get nodes -l workload-type=stateful --no-headers | wc -l)
  
  if [ $stateful_pods -ge $available_nodes ]; then
    echo "ERROR: Insufficient nodes for stateful pod rescheduling"
    exit 1
  fi
}
```

---