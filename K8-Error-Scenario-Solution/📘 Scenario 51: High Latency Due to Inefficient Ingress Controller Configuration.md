# 📘 Scenario 51: High Latency Due to Inefficient Ingress Controller Configuration

**Category**: Network Performance & Ingress Routing  
**Environment**: Kubernetes 1.20, AWS EKS with NGINX Ingress Controller  
**Impact**: 500ms+ latency increase, 30% error rate during peak traffic  

---

## Scenario Summary  
Inefficient Ingress routing configurations caused severe latency spikes and packet processing delays, degrading user experience and increasing error rates for external-facing applications.

---

## What Happened  
- **Routing complexity explosion**:  
  - 500+ ingress rules with complex regex path matching  
  - Default NGINX configuration with 100k+ `worker_connections` limit  
  - No caching or keep-alive optimizations enabled  
- **Performance degradation**:  
  - P95 latency increased from 50ms to 580ms  
  - NGINX worker processes at 100% CPU utilization  
  - `upstream request timed out` errors in application logs  
- **Configuration audit findings**:  
  - 80% of routes used regex matching instead of exact paths  
  - Missing HTTP/2 and gzip compression  
  - No rate limiting or connection pooling  

---

## Diagnosis Steps  

### 1. Monitor ingress controller metrics:
```sh
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/ingress-nginx/pods | jq '.items[] | .metadata.name + " " + (.containers[].usage.cpu | tonumber/1000000)'
# Showed 90%+ CPU utilization
```

### 2. Check NGINX configuration:
```sh
kubectl exec -n ingress-nginx deploy/nginx-ingress-controller -- nginx -T | head -100
# Revealed 500+ location blocks with regex
```

### 3. Analyze access logs for slow requests:
```sh
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=1000 | \
  awk '{print $NF}' | sort -n | tail -10
# Output: Processing times >500ms
```

### 4. Inspect ingress resource complexity:
```sh
kubectl get ingress -A -o yaml | yq '.items[].spec.rules[].http.paths | length' | sort -nr | head -5
# Showed one ingress with 150+ paths
```

---