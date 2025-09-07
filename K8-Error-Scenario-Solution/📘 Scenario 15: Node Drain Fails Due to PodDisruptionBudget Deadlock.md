# 📘 Scenario 15: Node Drain Fails Due to PodDisruptionBudget Deadlock

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
