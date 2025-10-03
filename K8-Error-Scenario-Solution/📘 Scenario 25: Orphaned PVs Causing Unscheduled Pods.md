# 📘 Scenario 25: Orphaned PVs Causing Unscheduled Pods

**Category**: Storage Management  
**Environment**: Kubernetes 1.20, vSphere CSI Driver 2.3  
**Impact**: 47 PVCs stuck in Pending state, blocking application deployments  

---
## Scenario Summary  
PersistentVolumes (PVs) left in `Released` state after pod deletions prevented new PVCs from binding, causing critical workloads to fail scheduling.

---

## What Happened  
- **Storage lifecycle gap**:  
  - PVs created with `persistentVolumeReclaimPolicy: Retain`  
  - No cleanup process for released volumes  
- **Provisioning failure**:  
  - New PVCs for same storage class failed with `waiting for first consumer to be created`  
  - vSphere CSI logs showed `volume already exists` errors  
- **Cascading effects**:  
  - StatefulSet pods stuck in `Pending`  
  - Database initialization jobs timed out  

---

## Diagnosis Steps  

### 1. Verify PVC status:
```sh
kubectl get pvc -A -o wide | grep -v Bound
# Showed multiple PVCs pending
```

### 2. Check PV inventory:
```sh
kubectl get pv --sort-by=.metadata.creationTimestamp \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.claimRef.name,RECLAIM:.spec.persistentVolumeReclaimPolicy
# Revealed 20+ PVs in Released state
```

### 3. Inspect storage provider:
```sh
kubectl logs -n kube-system ds/vsphere-csi-node -c vsphere-csi-driver | grep -A10 "already exists"
```

### 4. Audit reclaim policies:
```sh
kubectl get sc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.reclaimPolicy}{"\n"}{end}'
# Showed 80% volumes configured with Retain
```

---

## Root Cause  
**Storage lifecycle breakdown**:  
1. `Retain` policy required manual intervention  
2. CSI driver couldn't reprovision existing volumes  
3. No monitoring for orphaned PVs  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Delete orphaned PVs (after data backup)
kubectl get pv | grep Released | awk '{print $1}' | xargs -I{} kubectl delete pv {}

# 2. Retry pending PVCs
kubectl get pvc -A | grep Pending | awk '{print $1,$2}' | \
  xargs -n2 sh -c 'kubectl patch pvc -n $0 $1 -p '"'"'{"metadata":{"annotations":{"volume.beta.kubernetes.io/storage-class":"force-retry"}}'"'"'
```

### Long-term Solution:
```yaml
# StorageClass update
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vsphere-ssd
provisioner: csi.vsphere.vmware.com
reclaimPolicy: Delete  # Changed from Retain
volumeBindingMode: WaitForFirstConsumer
```

---

## Lessons Learned  
⚠️ **Released ≠ Recycled**: PVs require explicit cleanup  
⚠️ **Retain policies are dangerous**: Need automated follow-up  
⚠️ **Storage is stateful**: Must track across entire lifecycle  

---

## Prevention Framework  

### 1. Automated Cleanup
```yaml
# K8s Job to clean Released PVs (runs weekly)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pv-cleaner
spec:
  schedule: "0 3 * * 6"  # Saturday 3AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleaner
            image: bitnami/kubectl
            command:
            - /bin/sh
            - -c
            - |
              kubectl get pv --no-headers | \
                grep Released | \
                awk '{print $1}' | \
                xargs kubectl delete pv
```

### 2. Storage Policy
```yaml
# OPA/Gatekeeper constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sValidStorageClass
metadata:
  name: require-delete-policy
spec:
  match:
    kinds:
    - apiGroups: ["storage.k8s.io"]
      kinds: ["StorageClass"]
  parameters:
    allowedReclaimPolicies: ["Delete"]
```


### 3. Monitoring
```yaml
# Prometheus alerts
- alert: OrphanedPVs
  expr: count(kube_persistentvolume_status_phase{phase="Released"}) > 0
  for: 1h
  labels:
    severity: warning

- alert: PendingPVCs
  expr: count(kube_persistentvolumeclaim_status_phase{phase="Pending"}) > 5
  for: 30m
  labels:
    severity: critical
```

### 4. Documentation

## PV Cleanup Protocol
1. **Identify candidates**:
   ```sh
   kubectl get pv -o json | jq -r '.items[] | select(.status.phase=="Released") | .metadata.name'
   ```
2. **Backup data** (if needed):
   ```sh
   velero backup create pv-cleanup-$(date +%s) --include-resources persistentvolumes
   ```
3. **Release volumes**:
   ```sh
   kubectl delete pv <name> --wait=false
   ```

---
