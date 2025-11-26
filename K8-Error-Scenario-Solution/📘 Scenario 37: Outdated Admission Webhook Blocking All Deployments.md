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

### 3. Inspect webhook configuration:
```sh
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -o wide
# Identified problematic webhook
```

### 4. Verify webhook pod status:
```sh
kubectl -n webhook-ns get pods -l app=webhook
# Showed Running but TLS certificate expired
```

---

## Root Cause  
**Certificate lifecycle failure**:  
1. Manual certificate management without renewal automation  
2. No monitoring for certificate expiration  
3. Webhook configured without proper failure policies  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Temporarily disable webhook
kubectl patch mutatingwebhookconfiguration admission-webhook -p '{
  "webhooks": [{
    "name": "admission-webhook.example.com",
    "failurePolicy": "Ignore"
  }]
}'


# 2. Deploy critical workloads
kubectl apply -f emergency-patch/

# 3. Renew and redeploy webhook
kubectl create secret tls webhook-tls --cert=new.crt --key=new.key -n webhook-ns
kubectl rollout restart deployment/webhook -n webhook-ns
