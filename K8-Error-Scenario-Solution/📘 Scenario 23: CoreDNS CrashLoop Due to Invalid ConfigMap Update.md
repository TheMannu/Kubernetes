# 📘 Scenario 23: CoreDNS CrashLoop Due to Invalid ConfigMap Update

**Category**: Cluster Networking  
**Environment**: Kubernetes 1.23, GKE (Managed Control Plane)  
**Impact**: Cluster-wide DNS outage lasting 47 minutes  

---

## Scenario Summary  
A malformed CoreDNS ConfigMap update caused all DNS resolution to fail, breaking service discovery and pod-to-pod communication across the entire cluster.

---

## What Happened  
- **Config change**:  
  - Added rewrite rule with incorrect syntax: `rewrite stop name regex (.*)\.internal internal.svc.cluster.local`  
  - Missing plugin declaration in Corefile preamble  
- **Immediate impact**:  
  - CoreDNS pods entered `CrashLoopBackOff`  
  - `kubectl logs` showed `Corefile:5 - Error during parsing: Unknown directive 'rewrit'`  
- **Cascading failures**:  
  - Service mesh (Istio) sidecars failed health checks  
  - `kubelet` reported `node not ready` due to DNS timeouts  

---

## Diagnosis Steps  

### 1. Verify DNS availability:
```sh
kubectl run -it --rm dns-test --image=busybox -- nslookup kubernetes.default
# Error: `Server failure`
```

### 2. Check CoreDNS status:
```sh
kubectl -n kube-system get pods -l k8s-app=kube-dns
# Showed CrashLoopBackOff
```

### 3. Inspect pod logs:
```sh
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
# Output: `plugin/rewrite: unknown property "stop"`
```

### 4. Validate ConfigMap:
```sh
kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}' | grep -A5 rewrite
# Revealed malformed rewrite rule
```

---

## Root Cause  
**Configuration error**:  
1. Missing `rewrite` plugin in CoreDNS deployment  
2. Typo in rewrite directive (`rewrit` vs `rewrite`)  
3. No validation before applying changes  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Rollback ConfigMap
kubectl -n kube-system rollout undo configmap/coredns

# 2. Force pod restart
kubectl -n kube-system rollout restart deployment/coredns

# 3. Verify restoration
kubectl run -it --rm dns-test --image=busybox -- nslookup kubernetes.default
```

### Correct Configuration:
```corefile
# Valid rewrite rule
.:53 {
    rewrite name regex (.*)\.internal internal.svc.cluster.local
    # ... other plugins ...
}
```

---

## Lessons Learned  
⚠️ **DNS is critical infrastructure**: Breaks within seconds of bad config  
⚠️ **CoreDNS is unforgiving**: Silent until reload, then hard fails  
⚠️ **Managed ≠ foolproof**: GKE doesn't validate Corefile syntax  

---

## Prevention Framework  

### 1. Validation Workflow
```sh
# Pre-apply check using official image
docker run -i coredns/coredns:1.8.6 -conf - <<< "$(kubectl get configmap/coredns -n kube-system -o jsonpath='{.data.Corefile}')"
```

### 2. GitOps Safeguards
```yaml
# ArgoCD Application with pre-sync hook
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    syncOptions:
    - Validate=true
    hooks:
      preSync:
        - name: validate-coredns
          template: coredns-validator
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: CoreDNSDown
  expr: absent(up{job="kube-dns"} == 1)
  for: 1m
  labels:
    severity: critical
```

### 4. Change Management
```markdown
## CoreDNS Config Checklist
1. [ ] Test in staging cluster  
2. [ ] Validate with `coredns -conf`  
3. [ ] Document change in runbook  
4. [ ] Prepare rollback procedure  
```

---

**Key Metrics to Monitor**:  
- `coredns_dns_responses_total{rcode="SERVFAIL"}`  
- `kubelet_dns_errors`  
- `probe_dns_lookup_time_seconds`  
