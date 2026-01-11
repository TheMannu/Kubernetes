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
