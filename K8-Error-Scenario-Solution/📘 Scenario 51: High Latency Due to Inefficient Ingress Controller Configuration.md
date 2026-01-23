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

## Root Cause  
**Inefficient routing configuration**:  
1. Regex path matching instead of exact/prefix matching  
2. NGINX `location` blocks evaluated sequentially  
3. Missing performance optimizations (caching, keepalive, HTTP/2)  
4. Excessive ingress fragmentation (many small resources vs consolidated)  

---

## Fix/Workaround  

### Immediate Optimization:
```yaml
# Consolidated ingress with optimized routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: optimized-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "false"  # Disable regex by default
    nginx.ingress.kubernetes.io/configuration-snippet: |
      # Performance optimizations
      limit_conn perip 10;
      limit_conn perserver 100;
    nginx.ingress.kubernetes.io/proxy-buffering: "on"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
    nginx.ingress.kubernetes.io/proxy-buffers-number: "4"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1/users
        pathType: Prefix  # Changed from ImplementationSpecific
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /api/v1/products
        pathType: Exact   # Exact matching for performance
        backend:
          service:
            name: product-service
            port:
              number: 8080
```

### NGINX Configuration Tuning:
```yaml
# ConfigMap for performance tuning
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
data:
  worker-processes: "auto"
  worker-connections: "65536"
  keep-alive: "75"
  keep-alive-requests: "1000"
  upstream-keepalive-connections: "32"
  upstream-keepalive-timeout: "60"
  upstream-keepalive-requests: "100"
  http2-max-field-size: "16k"
  http2-max-header-size: "32k"
  proxy-buffer-size: "16k"
  proxy-buffers: "4 16k"
```

---

## Lessons Learned  
⚠️ **Regex is expensive**: Use `Exact` or `Prefix` path types when possible  
⚠️ **Location order matters**: NGINX evaluates location blocks sequentially  
⚠️ **Connection management**: Keep-alive and HTTP/2 dramatically improve performance  

---

## Prevention Framework  

### 1. Ingress Validation Pipeline
```sh
# Pre-apply ingress validation
validate_ingress() {
  local ingress_file=$1
  
  # Check for excessive regex usage
  local regex_count=$(grep -c "ImplementationSpecific\|use-regex.*true" $ingress_file)
  if [ $regex_count -gt 10 ]; then
    echo "WARNING: High regex usage ($regex_count instances) - consider Prefix/Exact matching"
  fi
```
  
### 2. Performance Monitoring
```yaml
# Prometheus alerts for ingress performance
- alert: IngressHighLatency
  expr: histogram_quantile(0.95, rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) > 0.5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Ingress P95 latency >500ms ({{ $value }} seconds)"

- alert: IngressHighErrorRate
  expr: rate(nginx_ingress_controller_requests{status=~"5.."}[5m]) / rate(nginx_ingress_controller_requests[5m]) > 0.05
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Ingress 5xx error rate >5%"

- alert: IngressWorkerSaturation
  expr: nginx_ingress_controller_nginx_process_connections_total / nginx_ingress_controller_nginx_process_connections_limit > 0.8
  for: 10m
  labels:
    severity: warning
```

### 3. Configuration Standards
```yaml
# Ingress template with performance best practices
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # Performance
    nginx.ingress.kubernetes.io/use-regex: "false"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
    nginx.ingress.kubernetes.io/proxy-buffering: "on"
    nginx.ingress.kubernetes.io/proxy-buffers-number: "4"
    
    # Security
    nginx.ingress.kubernetes.io/limit-connections: "100"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    
    # Observability
    nginx.ingress.kubernetes.io/enable-opentracing: "true"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Request-ID: $request_id";
```

### 4. Load Testing Framework
```yaml
# Vegeta load test for ingress validation
apiVersion: batch/v1
kind: Job
metadata:
  name: ingress-load-test
spec:
  template:
    spec:
      containers:
      - name: vegeta
        image: peterevans/vegeta
        command: ["sh", "-c"]
        args:
        - |
          echo "GET https://api.example.com/healthz" | \
          vegeta attack -rate=1000 -duration=30s -insecure | \
          vegeta report > /tmp/report.txt && \
          cat /tmp/report.txt
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

**Key Performance Optimizations**:  
- **Path Matching Priority**: Exact > Prefix > Regex  
- **Connection Pooling**: Upstream keep-alive connections  
- **Buffer Tuning**: Optimized for request/response sizes  
- **Compression**: Gzip for text-based responses  
- **Caching**: Static asset caching where applicable  
