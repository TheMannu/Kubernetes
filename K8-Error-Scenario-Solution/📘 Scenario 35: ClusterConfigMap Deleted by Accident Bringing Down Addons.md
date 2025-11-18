# 📘 Scenario 35: ClusterConfigMap Deleted by Accident Bringing Down Addons

**Category**: Cluster Configuration  
**Environment**: Kubernetes 1.24, Rancher 2.6  
**Impact**: Critical system addons (DNS, metrics, service mesh) failed for 45 minutes  

---

## Scenario Summary  
Accidental deletion of the `kube-root-ca.crt` ConfigMap caused widespread failures across system workloads that relied on it for TLS trust, breaking core cluster functionality.

---

## What Happened  
- **Accidental deletion**:  
  - User ran `kubectl delete cm kube-root-ca.crt -n kube-system` during cleanup  
  - No RBAC restrictions prevented the deletion  
- **Immediate impact**:  
  - CoreDNS pods failed with `configmap "kube-root-ca.crt" not found`  
  - Metrics-server, ingress controllers, and service mesh sidecars crashed  
  - Pods stuck in `CreateContainerConfigError` state  
- **Cascading effects**:  
  - Service discovery broken due to DNS failures  
  - HPA unable to scale due to missing metrics  
  - Monitoring alerts flooded due to component failures  

---

## Diagnosis Steps  

### 1. Identify failing pods:
```sh
kubectl get pods -A --field-selector status.phase!=Running
# Showed 20+ pods in CreateContainerConfigError
```

### 2. Check pod events:
```sh
kubectl describe pod -n kube-system coredns-<hash> | grep -A10 Events
# Output: "MountVolume.SetUp failed: configmap "kube-root-ca.crt" not found"
```

### 3. Verify ConfigMap existence:
```sh
kubectl get cm kube-root-ca.crt -n kube-system
# Error: Error from server (NotFound): configmaps "kube-root-ca.crt" not found
```

### 4. Audit recent changes:
```sh
kubectl get events -A --sort-by=.lastTimestamp | tail -20
# Showed deletion event
```

---
