# 📘 Scenario #34: Etcd Disk Full Crashing the Cluster

**Category**: Control Plane Stability  
**Environment**: Kubernetes 1.21, Self-managed with Local etcd  
**Impact**: Complete cluster outage lasting 3+ hours  

---

## Scenario Summary  
The etcd database exhausted all available disk space due to unmanaged growth of custom resources and lack of maintenance, causing complete control plane failure.

---

## What Happened  
- **Resource explosion**:  
  - Custom controller creating 10,000+ ConfigMaps daily  
  - No resource quotas or cleanup mechanisms  
- **Storage exhaustion**:  
  - `/var/lib/etcd` reached 100% capacity  
  - etcd logs showed `"mvcc: database space exceeded"`  
  - API server failed with `"etcdserver: no space"` errors  
- **Cascading failures**:  
  - Control plane components lost leader election  
  - Worker nodes marked `NotReady` after failed heartbeats  

---

## Diagnosis Steps  

### 1. Verify etcd status:
```sh
kubectl get pods -n kube-system -l component=etcd
# Showed CrashLoopBackOff
```

### 2. Check disk usage:
```sh
df -h /var/lib/etcd
# Output: 100% used
```

### 3. Inspect etcd logs:
```sh
journalctl -u etcd --no-pager -n 50 | grep -i "space\|quota"
# Output: "etcdserver: mvcc: database space exceeded"
```

### 4. Analyze database size:
```sh
etcdctl endpoint status -w table
# Showed 8GB database on 10GB disk
```

---

## Root Cause  
**Storage management failure**:  
1. No automatic compaction of historical revisions  
2. Missing disk space monitoring  
3. Unbounded custom resource creation  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Free up space temporarily
sudo find /var/lib/etcd -name "*.snap.*" -mtime +7 -delete

# 2. Compact etcd history
etcdctl compact $(etcdctl endpoint status -w json | jq -r '.[].Status.header.revision')

# 3. Defragment database
etcdctl defrag --cluster

# 4. Restart etcd
sudo systemctl restart etcd
```

### Long-term Solution:
```yaml
# etcd configuration
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --auto-compaction-mode=revision
    - --auto-compaction-retention=1000
    - --quota-backend-bytes=8589934592  # 8GB limit
    - --max-request-bytes=1572864
```

---

## Lessons Learned  
⚠️ **Etcd is stateful**: Requires proactive disk management  
⚠️ **Silent growth**: Custom resources can explode unnoticed  
⚠️ **Hard limits matter**: Without quotas, etcd will consume all space  

---

## Prevention Framework  

### 1. Automated Maintenance
```yaml
# CronJob for etcd maintenance
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-maintenance
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: maintenance
            image: bitnami/etcd:3.5
            command:
            - /bin/sh
            - -c
            - |
              etcdctl compact $(etcdctl endpoint status -w json | jq -r '.[].Status.header.revision')
              etcdctl defrag --cluster
```

### 2. Resource Quotas
```yaml
# Namespace quotas for CR-heavy workloads
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-count
  namespace: custom-operators
spec:
  hard:
    configmaps: "1000"
    secrets: "1000"
    customresourcedefinitions: "50"
```
