# 📘 Scenario 9: CoreDNS CrashLoop on Startup  

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.24, DigitalOcean  
**Impact**: Cluster-wide DNS resolution failure  

---

## Scenario Summary  
CoreDNS pods entered a `CrashLoopBackOff` state due to an invalid `Corefile` configuration, breaking DNS resolution across all cluster workloads.  

---

## What Happened  
- **Custom `Corefile` update**:  
  - A new `rewrite` rule was added with incorrect syntax  
  - Applied via `kubectl edit configmap/coredns -n kube-system`  
- **Immediate failure**:  
  - CoreDNS pods crashed on startup (`Error in Corefile: invalid directive`)  
  - Cluster services began failing with `NXDOMAIN` errors  
- **Cascading effects**:  
  - `kubectl` commands slowed due to DNS timeouts  
  - Service mesh (Istio) sidecars failed health checks  

---

## Diagnosis Steps  

### 1. Check CoreDNS pod status:
```sh
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Output showed CrashLoopBackOff
```

### 2. Inspect pod logs:
```sh
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# Error: "Corefile:5 - Error during parsing: Unknown directive 'rewrit'"
```

### 3. Verify ConfigMap:
```sh
kubectl get configmap/coredns -n kube-system -o yaml | grep -A10 "Corefile"
# Revealed malformed rewrite rule:
# rewrit name exact foo.bar internal.svc.cluster.local
```

### 4. Validate Corefile syntax (offline):
```sh
docker run -i coredns/coredns:1.8.6 -conf - <<< "$(kubectl get configmap/coredns -n kube-system -o jsonpath='{.data.Corefile}')"
```
---

## Root Cause  
**Configuration error**:  
- Typo in directive (`rewrit` vs `rewrite`)  
- Missing required plugin (`rewrite` not in CoreDNS image)  
- No validation before applying changes  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# Revert to previous ConfigMap version
kubectl rollout undo configmap/coredns -n kube-system

# Force CoreDNS rollout
kubectl rollout restart deployment/coredns -n kube-system
```

### Permanent Solution:
1. **Add validation workflow**:
   ```sh
   # Pre-apply check using coredns/corerc
   docker run -i coredns/corerc:latest validate < corefile-new.cfg
   ```

2. **Implement GitOps**:
   ```yaml
   # ArgoCD Application with kustomize
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   spec:
     syncPolicy:
       automated:
         selfHeal: true  # Auto-revert broken changes
   ```

---

## Lessons Learned  
⚠️ **DNS is critical infrastructure**: Even syntax errors cause cluster-wide outages  
⚠️ **ConfigMaps need version control**: `kubectl edit` is dangerous without backups  

---

## Prevention Framework  

### 1. Validation Safeguards
```sh
# Pre-commit hook example (.git/hooks/pre-commit)
#!/bin/sh
docker run -i coredns/coredns:${CORE_DNS_VERSION} -conf - < corefile.cfg || {
  echo "Corefile validation failed"
  exit 1
}
```

