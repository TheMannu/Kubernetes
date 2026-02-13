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
