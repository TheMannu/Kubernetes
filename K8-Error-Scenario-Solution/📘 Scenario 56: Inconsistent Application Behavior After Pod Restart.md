# 📘 Scenario 56: Inconsistent Application Behavior After Pod Restart

**Category**: Stateful Application Design  
**Environment**: Kubernetes 1.20, GKE, Stateless-by-default expectations  
**Impact**: Data inconsistency across users, revenue loss from corrupted sessions, 4-hour debugging cycle  

---

## Scenario Summary  
An application storing session state and user preferences in ephemeral container storage exhibited unpredictable behavior after pod restarts, resulting in data loss, inconsistent user experiences, and hard-to-reproduce bugs.

---

## What Happened  
- **Trigger event**:  
  - Node maintenance triggered pod rescheduling  
  - 3 of 12 application pods were terminated and recreated  
- **Observed symptoms**:  
  - Users reported shopping carts randomly emptying  
  - Some users saw outdated profile information  
  - Admin dashboards showed inconsistent analytics  
  - Support tickets spiked 300% within 1 hour  
- **Root analysis**:  
  - Application stored user session data in `/tmp` (ephemeral storage)  
  - No leader election for stateful operations  
  - Caches not rebuilt after pod restart  
  - No health check for state readiness  

---

## Diagnosis Steps  

### 1. Check pod restart history:
```sh
kubectl get pods -l app=frontend -n ecommerce | grep -v Running
kubectl describe pod frontend-abc123 -n ecommerce | grep -A10 "Events"
# Showed 3 pods restarted in last 30 minutes
```

### 2. Inspect application logs:
```sh
kubectl logs -l app=frontend -n ecommerce --tail=100 | grep -i "session\|cache\|state"
# Output: "Failed to load session data: file not found"
```

### 3. Verify storage configuration:
```sh
kubectl get deployment frontend -n ecommerce -o yaml | yq '.spec.template.spec.volumes'
# No persistent volume claims configured
kubectl exec -it frontend-abc123 -n ecommerce -- ls -la /tmp/sessions
# Showed local session files (lost on restart)
```

### 4. Check state synchronization:
```sh
kubectl logs frontend-xyz789 -n ecommerce | grep -i "leader"
# No leader election logs found
```

---

## Root Cause  
**Ephemeral state anti-pattern**:  
1. Session data stored in container filesystem (non-persistent)  
2. No external state store (Redis, database) for shared state  
3. Missing startup health checks to verify state readiness  
4. Load balancer sent traffic to pods before cache warmup  

---

## Fix/Workaround  

### Immediate Mitigation:
```sh
# 1. Scale down to prevent further inconsistent state
kubectl scale deployment frontend -n ecommerce --replicas=0

# 2. Re-deploy with external state store
helm upgrade frontend ./helm-chart \
  --set redis.enabled=true \
  --set session.storage.type=redis
```


### Long-term Solution:
```yaml
# Deployment with external state dependencies
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
spec:
  replicas: 6
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: app
        image: myapp/frontend:2.1.0
        env:
        - name: REDIS_HOST
          value: redis-master.ecommerce.svc.cluster.local
        - name: SESSION_STORAGE
          value: redis
        - name: CACHE_WARMUP_ENABLED
          value: "true"
        volumeMounts:
        - name: tmp
          mountPath: /tmp  # Only for non-critical data
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 30  # Wait for cache warmup
          periodSeconds: 10
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
      volumes:
      - name: tmp
        emptyDir: {}
---