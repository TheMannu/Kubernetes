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


## Root Cause  
**Incomplete backup strategy**:  
1. Velero configured without `--snapshot-volumes=true`  
2. No coordination between etcd backups and CSI snapshots  
3. External secrets (Vault) not included in backup scope  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# 1. Recreate critical secrets from backup
kubectl create secret generic registry-key \
--from-file=./backups/secrets/registry-key.json

# 2. Provision replacement PVs
velero restore create --from-backup $PVC_BACKUP --include-resources persistentvolumeclaims
```

### Full Restoration:
```sh
# 3. Redeploy applications with corrected dependencies
kubectl rollout restart deploy -n production
```

---

## Lessons Learned  
⚠️ **etcd is only part of the picture**: Application state spans multiple systems  
⚠️ **Dependencies matter**: Objects without their dependencies create "zombie" resources  
⚠️ **Silent exclusions**: Backup tools often skip resources by default  

---

## Prevention Framework  

### 1. Holistic Backup Policy
```yaml
# velero-backup.yaml
apiVersion: velero.io/v1
kind: Backup
spec:
  includedResources:
  - '*'
  excludedResources:
  - nodes,events,events.events.k8s.io
  snapshotVolumes: true
  ttl: 720h
  hooks:
    resources:
    - name: pre-vault-backup
      pre:
        - exec:
            container: vault-agent
            command: ["/bin/sh", "-c", "vault operator raft snapshot save /backups/vault.snap"]
```


### 2. Restore Validation
```sh
# Post-restore checklist
validate_restore() {
  kubectl wait --for=condition=Ready pods --all --timeout=10m
  kubectl get pvc -A | grep -v Bound && echo "Unbound PVCs detected"
}
```

### 3. Backup Monitoring
```yaml
# Prometheus alerts
- alert: IncompleteBackup
  expr: velero_backup_success_total == 0 or velero_volume_snapshot_failure_total > 0
  for: 15m
  labels:
    severity: critical
```

### 4. Disaster Recovery Drills
```markdown
Quarterly Test Plan:
1. [ ] Simulate etcd corruption  
2. [ ] Restore from Velero + CSI snapshots  
3. [ ] Validate application connectivity  
4. [ ] Measure time-to-recovery (TTR)  
```

---

**Key Resources to Backup**:  
- **etd**: All Kubernetes objects  
- **CSI**: Persistent Volumes  
- **External**: Secrets (Vault/HashiCorp), Container Images  
- **Configuration**: CNI, CSI, Ingress controllers  


**Restore Verification Tools**:  
```sh
# Check object consistency
kubectl get all,cm,secret,pvc -A --no-headers | wc -l
# Compare with backup
velero backup describe $BACKUP --details | grep "Resource"

# Validate storage
kubectl exec -it test-pod -- df -h /data
```
**Velero Best Practices**:  
```yaml
# Schedule with hooks
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-full
spec:
  schedule: "@daily"
  template:
    hooks:
      resources:
      - name: freeze-db
        pre:
        - exec:
            container: db
            command: ["/bin/sh", "-c", "flush tables with read lock"]
        post:
        - exec:
            container: db
            command: ["/bin/sh", "-c", "unlock tables"]
```
---
---