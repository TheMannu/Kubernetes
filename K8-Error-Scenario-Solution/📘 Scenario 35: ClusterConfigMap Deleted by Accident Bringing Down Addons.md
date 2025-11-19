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

## Root Cause  
**Missing resource protection**:  
1. No RBAC restrictions on critical ConfigMaps  
2. Overly permissive cluster role bindings  
3. Missing admission controller protections  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Recreate the ConfigMap from backup cluster
kubectl get cm kube-root-ca.crt -n kube-system --context=backup-cluster -o yaml | \
  kubectl apply -f -

# 2. Restart affected deployments
kubectl rollout restart deployment -n kube-system coredns metrics-server


# 3. Verify recovery
kubectl get pods -n kube-system -l k8s-app=kube-dns --watch
```

### Manual Recreation (if no backup):
```yaml
# kube-root-ca.crt recreation
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-root-ca.crt
  namespace: kube-system
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    # Copy from /etc/kubernetes/pki/ca.crt on control plane
    -----END CERTIFICATE-----
```

---

## Lessons Learned  
⚠️ **Some resources are critical**: Cannot be recreated from application logic  
⚠️ **RBAC is essential**: Must protect system namespaces  
⚠️ **Dependencies are hidden**: Many components rely on implicit resources  

---
