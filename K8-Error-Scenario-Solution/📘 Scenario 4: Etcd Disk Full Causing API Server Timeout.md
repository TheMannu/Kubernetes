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
