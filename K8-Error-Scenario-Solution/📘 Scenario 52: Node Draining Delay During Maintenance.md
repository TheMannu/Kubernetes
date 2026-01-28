# 📘 Scenario 52: Node Draining Delay During Maintenance

**Category**: Cluster Maintenance & Reliability  
**Environment**: Kubernetes 1.21, GKE with Stateful Workloads  
**Impact**: Extended maintenance window, delayed node recycling, increased cloud costs  

---

## Scenario Summary  
Strict PodDisruptionBudget (PDB) configurations prevented timely node draining during maintenance, causing prolonged downtime and operational delays.

---

## What Happened  
- **Planned node maintenance**:  
  - GKE node auto-upgrade triggered for security patches  
  - 5-node cluster with 3-node Cassandra ring  
- **Draining bottlenecks**:  
  - `kubectl drain` hung for 45+ minutes  
  - PDB `minAvailable: 3` for Cassandra with only 3 replicas  
  - Storage-attached pods with `terminationGracePeriodSeconds: 300`  
  - No healthy pods ready to replace terminating ones  
- **Impact**:  
  - Maintenance window extended by 2 hours  
  - Cascading delays for subsequent node rotations  
  - SLA violations for stateful services  

---

## Diagnosis Steps  

### 1. Check node draining status:
```sh
kubectl get nodes
kubectl describe node <node-being-drained> | grep -A10 "Conditions"
# Showed SchedulingDisabled but pods still running
```

### 2. Investigate PDB violations:
```sh
kubectl get pdb -A
kubectl describe pdb cassandra-pdb -n database
# Output: Allowed disruptions: 0
```

### 3. Analyze pod termination:
```sh
kubectl get pods -n database -o wide | grep -E "Terminating|Pending"
kubectl describe pod cassandra-0 -n database | grep -A10 "Events"
# Showed waiting for volume detachment
```

### 4. Check pod disruption readiness:
```sh
kubectl get pods -n database -l app=cassandra -o json | \
  jq -r '.items[] | .metadata.name + " " + (.status.conditions[] | select(.type=="Ready") | .status)'
# Output: 1/3 pods not ready
```

---

## Root Cause  
**Overly conservative availability guarantees**:  
1. PDB `minAvailable` equal to replica count → zero disruption allowance  
2. Stateful pods with long termination grace periods  
3. No pod readiness for replacement scheduling  
4. Storage volume detachment delays  

---

## Fix/Workaround  

### Emergency Maintenance Acceleration:
```sh
# 1. Temporarily relax PDB (if SLA allows)
kubectl patch pdb cassandra-pdb -n database -p '{"spec":{"minAvailable":2}}'

# 2. Force delete stuck pods (after data verification)
kubectl delete pod cassandra-0 -n database --grace-period=0 --force

# 3. Complete node drain
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=600s
```

### Long-term Solution:
```yaml
# Updated PDB with buffer for maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cassandra-pdb
  namespace: database
spec:
  minAvailable: 2  # Changed from 3 (allows 1 pod disruption)
  selector:
    matchLabels:
      app: cassandra
---

# Pod configuration with faster failover
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # Reduced from 300
      containers:
      - name: cassandra
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "nodetool drain && sleep 30"]
```

---

## Lessons Learned  
⚠️ **PDB math matters**: `minAvailable = replicas` creates hard locks  
⚠️ **Grace periods compound delays**: Long terminations block node recycling  
⚠️ **Stateful workloads need special handling**: Storage and quorum considerations  

---

## Prevention Framework  

### 1. PDB Validation Rules
```sh
# Pre-apply PDB validation
validate_pdb() {
  local namespace=$1
  local pdb_name=$2
  
  local min_available=$(kubectl get pdb $pdb_name -n $namespace -o jsonpath='{.spec.minAvailable}')
  local replicas=$(kubectl get deploy -n $namespace -l app=$(kubectl get pdb $pdb_name -n $namespace -o jsonpath='{.spec.selector.matchLabels.app}') -o jsonpath='{.items[].spec.replicas}')
  

  if [ "$min_available" = "$replicas" ]; then
    echo "ERROR: PDB minAvailable ($min_available) equals replica count ($replicas) - will block all disruptions"
    exit 1
  fi
}
```

### 2. Maintenance Readiness Checks
```yaml
# Pre-maintenance validation Job
apiVersion: batch/v1
kind: Job
metadata:
  name: maintenance-readiness-check
spec:
  template:
    spec:
      containers:
      - name: checker
        image: bitnami/kubectl
        command: ["/bin/sh", "-c"]
        args:
        - |
          # Check PDB allowances
          kubectl get pdb -A -o json | jq -r '.items[] | select(.status.disruptionsAllowed == 0) | .metadata.namespace + "/" + .metadata.name'
          
          # Check pod distribution
          for node in $(kubectl get nodes -o name); do
            pod_count=$(kubectl get pods -A --field-selector spec.nodeName=${node#node/} --no-headers | wc -l)
            echo "Node ${node#node/} has $pod_count pods"
          done
          
          # Verify replacement capacity
          available_nodes=$(kubectl get nodes --no-headers | grep -c Ready)
          needed_nodes=$(( $(kubectl get pods -A --no-headers | wc -l) / 30 + 1 ))  # 30 pods per node
          if [ $available_nodes -le $needed_nodes ]; then
            echo "ERROR: Insufficient node capacity for maintenance"
            exit 1
          fi
```

### 3. Monitoring & Alerting
```yaml
# Prometheus alerts for maintenance readiness
- alert: PDBBlocksDisruptions
  expr: kube_poddisruptionbudget_status_disruptions_allowed == 0
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "PDB {{ $labels.namespace }}/{{ $labels.poddisruptionbudget }} allows zero disruptions"

- alert: NodeDrainStalled
  expr: time() - kube_node_spec_unschedulable > 1800
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Node {{ $labels.node }} has been unschedulable for >30 minutes"

- alert: LongTerminatingPods
  expr: time() - kube_pod_deletion_timestamp > 300
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pods terminating for >5 minutes"
```

### 4. Maintenance Automation
```yaml
# Automated node rotation with safety checks
apiVersion: batch/v1
kind: CronJob
metadata:
  name: node-rotation
spec:
  schedule: "0 2 * * 0"  # Sundays at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: rotator
            image: bitnami/kubectl
            command: ["/bin/sh", "-c"]
            args:
            - |
              # Select oldest node
              NODE=$(kubectl get nodes --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[0].metadata.name}')
              
              # Verify PDB allowances
              if kubectl get pdb -A -o json | jq -r '.items[] | select(.status.disruptionsAllowed == 0)' | grep -q .; then
                echo "PDB violations detected - aborting"
                exit 1
              fi
              