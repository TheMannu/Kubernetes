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