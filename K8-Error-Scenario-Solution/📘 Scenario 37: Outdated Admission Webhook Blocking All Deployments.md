# 📘 Scenario 37: Outdated Admission Webhook Blocking All Deployments

**Category**: Cluster Security & Operations  
**Environment**: Kubernetes 1.25, Self-hosted with Custom Admission Webhooks  
**Impact**: Complete deployment freeze for 3+ hours  

---

## Scenario Summary  
An expired TLS certificate on a mutating admission webhook caused all resource creation operations to fail, effectively freezing cluster operations and preventing any new deployments.

---

## What Happened  
- **Certificate expiration**:  
  - Webhook TLS certificate expired at 02:00 UTC  
  - No automatic certificate rotation configured  
- **Cluster-wide impact**:  
  - `kubectl apply` commands failed with `Internal error occurred: failed calling webhook`  
  - API server logs showed `x509: certificate has expired or is not yet valid`  
  - Emergency deployments for security patches blocked  
- **Failure mode**:  
  - Webhook configured as `fail-closed` (default behavior)  
  - API server couldn't reach webhook endpoint due to TLS errors  

---

## Diagnosis Steps  

### 1. Test resource creation:
```sh
kubectl create deployment test --image=nginx --dry-run=client -o yaml | kubectl apply -f -
# Error: "Internal error occurred: failed calling webhook"
```

### 2. Check API server logs:
```sh
kubectl -n kube-system logs -l component=kube-apiserver --tail=100 | grep -i webhook
# Output: "failed calling webhook: Post https://webhook-service.webhook-ns.svc:443/mutate?timeout=10s: x509: certificate expired"
```
