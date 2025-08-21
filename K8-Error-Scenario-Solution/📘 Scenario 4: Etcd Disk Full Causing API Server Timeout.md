# 📘 Scenario 4: Etcd Disk Full Causing API Server Timeout

**Category**: Cluster Management  
**Environment**: Kubernetes v1.25, Bare-metal cluster  

---

## Scenario Summary  
Cluster API server became unresponsive due to etcd running out of disk space from accumulated WAL logs and uncompacted revisions.

---

## What Happened  
- Cluster began **failing API requests** with timeout errors  
- **Etcd logs** showed `"mvcc: database space exceeded"` errors  
- **API server logs** indicated failed storage operations (`"etcdserver: request timed out"`)  
- Cluster operations (including `kubectl` commands) became unreliable  

---

## Diagnosis Steps  
1. **Verified disk space** on etcd nodes:  
   ```sh
   df -h /var/lib/etcd
   ```
2. **Analyzed etcd storage usage**:  
   ```sh
   sudo du -sh /var/lib/etcd/member/
   ```
3. **Checked etcd metrics**:  
   ```sh
   etcdctl endpoint status --write-out=table
   ```
4. **Identified root issues**:  
   - **No compaction** of historical revisions  
   - **Accumulated WAL logs** and snapshots  
   - **No disk space alerts** configured  

---

## Root Cause  
**Storage exhaustion due to**:  
1. Lack of **automatic compaction** of old revisions  
2. No **routine defragmentation** of etcd database  
3. Insufficient **disk space monitoring**  

---

## Fix/Workaround  

### Immediate Recovery:
1. **Compact old revisions** (replace `<rev>` with current revision):  
   ```sh
   etcdctl compact $(etcdctl endpoint status --write-out=json | jq -r '.[].Status.header.revision')
   ```
2. **Defragment etcd database**:  
   ```sh
   etcdctl defrag --cluster
   ```
3. **Clean up disk space**:  
   ```sh
   # Remove old snapshots/WAL logs (if safe)
   sudo find /var/lib/etcd/member/wal -type f -mtime +7 -delete
   ```

### Long-term Solution:
- **Increase etcd volume size**  
- **Configure automatic maintenance** (see prevention below)  

---

## Lessons Learned  
⚠️ **Etcd is stateful**: Requires proactive disk management.  
⚠️ **Silent failures**: API server errors may not clearly indicate etcd issues.  

---