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

# 4. Restore webhook enforcement
kubectl patch mutatingwebhookconfiguration admission-webhook -p '{
  "webhooks": [{
    "name": "admission-webhook.example.com", 
    "failurePolicy": "Fail"
  }]
}'
```

### Long-term Solution:
```yaml
# cert-manager Issuer for automatic renewal
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: webhook-ca
  namespace: webhook-ns
spec:
  ca:
    secretName: webhook-ca-secret
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: webhook-tls
  namespace: webhook-ns
spec:
  secretName: webhook-tls
  issuerRef:
    name: webhook-ca
    kind: Issuer
  dnsNames:
  - webhook-service.webhook-ns.svc
  - webhook-service.webhook-ns.svc.cluster.local
```

---

## Lessons Learned  
⚠️ **Webhooks are single points of failure**: Can block entire cluster operations  
⚠️ **TLS certificates have hard deadlines**: Must be proactively managed  
⚠️ **Failure policies matter**: `Fail` vs `Ignore` has major operational impact  

---

## Prevention Framework  

### 1. Certificate Automation
```yaml
# cert-manager configuration with alerts
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: webhook-tls
  annotations:
    # Alert 30 days before expiry
    cert-manager.io/alert-before-expiry: "720h"
spec:
  renewBefore: 240h  # Renew 10 days before expiry
  duration: 2160h    # 90-day certificates
```

### 2. Webhook Failure Policy
```yaml
# Safe webhook configuration
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: safe-webhook
webhooks:
- name: safe-webhook.example.com
  failurePolicy: Ignore  # Fail open during outages
  sideEffects: None
  timeoutSeconds: 5
  clientConfig:
    service:
      name: webhook-service
      namespace: webhook-ns
      path: /mutate
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: WebhookCertificateExpiring
  expr: certmanager_certificate_expiration_timestamp_seconds - time() < 86400 * 30  # 30 days
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Webhook certificate expiring in {{ $value | humanizeDuration }}"

- alert: WebhookCallFailures
  expr: rate(apiserver_admission_webhook_rejection_count[5m]) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Webhook rejecting requests ({{ $value }} errors/min)"
```

### 4. Deployment Safeguards
```sh
# Pre-flight webhook health check
check_webhook_health() {
  local response=$(curl -s -o /dev/null -w "%{http_code}" \
    https://webhook-service.webhook-ns.svc:443/healthz --connect-timeout 5)
  if [ "$response" != "200" ]; then
    echo "ERROR: Webhook health check failed"
    exit 1
  fi
}
```

---

**Key Webhook Best Practices**:  
- Use `failurePolicy: Ignore` for non-critical validations  
- Set reasonable `timeoutSeconds` (5-10s maximum)  
- Implement comprehensive health checks  
- Monitor certificate expiration proactively  
- Test webhook failure scenarios regularly  
