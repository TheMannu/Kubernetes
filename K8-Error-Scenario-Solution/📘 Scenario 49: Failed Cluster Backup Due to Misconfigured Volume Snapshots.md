# 📘 Scenario 49: Failed Cluster Backup Due to Misconfigured Volume Snapshots

**Category**: Disaster Recovery & Storage  
**Environment**: Kubernetes 1.21, AWS EKS with EBS CSI Driver  
**Impact**: Critical production backup failure, risking data loss for RPO of 24 hours  

---

## Scenario Summary  
A misconfigured EBS CSI snapshot driver caused complete backup failures, leaving the cluster without viable restore points for critical stateful workloads.

---

## What Happened  
- **Scheduled backup execution**:  
  - Velero scheduled backup triggered at 02:00 UTC  
  - Attempted to snapshot 150+ EBS volumes across 3 AZs  
- **Backup failure symptoms**:  
  - Velero logs showed `Failed to create snapshot: invalid request`  
  - CSI driver logs: `invalid volume snapshot class specified`  
  - 0/152 snapshots completed successfully  
- **Configuration issues**:  
  - VolumeSnapshotClass referenced non-existent IAM role  
  - StorageClass `allowVolumeExpansion: true` but snapshot class incompatible  
  - Missing tags for snapshot lifecycle management  

---

## Diagnosis Steps  

### 1. Check Velero backup status:
```sh
velero backup describe latest-backup --details
# Output: Phase: Failed, Errors: 152, Snapshots Attempted: 0
```

### 2. Inspect Velero logs:
```sh
kubectl logs -n velero -l component=velero --tail=100 | grep -i "snapshot\|failed"
# Output: "error creating snapshot: rpc error: code = InvalidArgument desc = Invalid volume snapshot class"
```

### 3. Verify VolumeSnapshotClass configuration:
```sh
kubectl get volumesnapshotclass -o yaml | yq '.items[].driver'
# Output: ebs.csi.aws.com but missing required parameters
```

### 4. Check CSI driver permissions:
```sh
kubectl logs -n kube-system -l app=ebs-csi-controller --tail=50 | grep -i "permission\|access"
# Output: "AccessDenied: User arn:aws:iam::123456789:role/ebs-csi-driver is not authorized"
```

---

## Root Cause  
**CSI driver configuration mismatch**:  
1. VolumeSnapshotClass referenced wrong CSI driver (`ebs.csi.aws.com` vs correct `ebs.csi.aws.com/v1`)  
2. IAM role missing `ec2:CreateSnapshot` permission  
3. StorageClass and SnapshotClass compatibility issues  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Create manual EBS snapshots via AWS CLI (emergency)
aws ec2 create-snapshot --volume-id vol-abc123 --description "Emergency backup $(date +%Y%m%d)"

# 2. Fix IAM permissions
aws iam attach-role-policy --role-name ebs-csi-driver \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# 3. Update VolumeSnapshotClass
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
  annotations:
    velero.io/csi-volumesnapshot-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Retain
parameters:
  tagSpecification_1: "ResourceType=snapshot,Tags=[{Key=backup,Value=velero}]"
EOF
```

### Long-term Solution:
```yaml
# Complete Velero + CSI configuration
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: aws
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: my-velero-backups
  config:
    region: us-west-2
---
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: aws
  namespace: velero
spec:
  provider: aws
  config:
    region: us-west-2
    profile: default
```

---

## Lessons Learned  
⚠️ **Backup failures are silent disasters**: Only discovered during restore attempts  
⚠️ **IAM permissions are critical**: CSI drivers need explicit snapshot permissions  
⚠️ **Configuration drift happens**: Snapshot classes must match storage classes  

---

## Prevention Framework  

### 1. Backup Validation Pipeline
```sh
# Automated backup validation script
validate_backup() {
  local backup_name=$1
  
  # Check backup completion
  if ! velero backup describe $backup_name --details | grep -q "Phase: Completed"; then
    echo "ERROR: Backup $backup_name failed"
    exit 1
  fi
  
  # Verify snapshot count matches PVC count
  local pvc_count=$(kubectl get pvc -A --no-headers | wc -l)
  local snapshot_count=$(velero backup describe $backup_name --details | grep "Snapshot ID" | wc -l)
  
  if [ $pvc_count -ne $snapshot_count ]; then
    echo "ERROR: Snapshot count mismatch (PVCs: $pvc_count, Snapshots: $snapshot_count)"
    exit 1
  fi
}
```

### 2. IAM Policy Management
```json
// Minimal EBS CSI driver IAM policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot",
        "ec2:DescribeSnapshots",
        "ec2:ModifySnapshotAttribute",
        "ec2:DescribeVolumes",
        "ec2:DescribeTags",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3. Monitoring & Alerting
```yaml
# Prometheus alerts for backup health
- alert: BackupFailure
  expr: velero_backup_failure_total > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Velero backup failed ({{ $value }} failures)"

- alert: MissingSnapshots
  expr: velero_volume_snapshot_attempt_total - velero_volume_snapshot_success_total > 0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Volume snapshot failures detected"

- alert: OldBackup
  expr: time() - velero_backup_last_successful_timestamp_seconds > 86400
  for: 1h
  labels:
    severity: critical
  annotations:
    summary: "No successful backup in 24 hours"
```

### 4. Configuration Testing
```yaml
# Pre-production backup test job
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-test
  namespace: velero
spec:
  template:
    spec:
      containers:
      - name: tester
        image: velero/velero:v1.9.0
        command: ["/bin/sh", "-c"]
        args:
        - |
          # Create test PVC
          kubectl apply -f test-pvc.yaml
          
          # Run test backup
          velero backup create test-backup --include-namespaces=backup-test
          
          # Verify backup
          velero backup describe test-backup --details | grep -q "Phase: Completed" || exit 1
          
          # Test restore
          velero restore create --from-backup test-backup
          
          # Cleanup
          velero backup delete test-backup --confirm
```

---

**Key Backup Components**:  
- **CSI Snapshot Controller**: Manages VolumeSnapshot CRDs  
- **VolumeSnapshotClass**: Defines snapshot parameters and driver  
- **Velero Restic**: For file-level backups of volumes without CSI  
- **StorageClass Compatibility**: Must support snapshots  

**Debugging Tools**:  
```sh
# Check CSI driver health
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver

# Verify VolumeSnapshot CRDs exist
kubectl get crd | grep volumesnapshot

# Test snapshot creation manually
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: test-pvc
EOF

# Check AWS permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:role/ebs-csi-driver \
  --action-names ec2:CreateSnapshot
```

**Backup Configuration Checklist**:  
```markdown
1. [ ] CSI driver installed and healthy  
2. [ ] VolumeSnapshotClass configured with correct driver  
3. [ ] IAM role has snapshot permissions  
4. [ ] StorageClass supports snapshots (`allowVolumeExpansion: true`)  
5. [ ] Velero VolumeSnapshotLocation configured  
6. [ ] Regular backup validation tests  
7. [ ] Monitoring alerts configured  
8. [ ] Disaster recovery runbook documented  
```

**Emergency Manual Backup Procedure**:  
```sh
# If automated backup fails completely
for pvc in $(kubectl get pvc -A -o jsonpath='{.items[*].spec.volumeName}'); do
  volume_id=$(aws ec2 describe-volumes --filters Name=tag:kubernetes.io/created-for/pvc/name,Values=$pvc --query 'Volumes[0].VolumeId')
  aws ec2 create-snapshot --volume-id $volume_id --tag-specifications 'ResourceType=snapshot,Tags=[{Key=EmergencyBackup,Value=true}]'
done

# Export critical ConfigMaps and Secrets
kubectl get cm -A -o yaml > configmaps-backup.yaml
kubectl get secret -A -o yaml > secrets-backup.yaml
```