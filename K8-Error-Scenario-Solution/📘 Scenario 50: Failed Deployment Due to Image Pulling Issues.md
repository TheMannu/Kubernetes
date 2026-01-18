# 📘 Scenario 50: Failed Deployment Due to Image Pulling Issues

**Category**: Container Registry & Authentication  
**Environment**: Kubernetes 1.22, Private Docker Registry (Harbor)  
**Impact**: Production deployment failure, 2-hour service disruption  

---

## Scenario Summary  
Deployments failed across multiple clusters due to misconfigured image pull secrets, preventing pods from pulling container images from a private Docker registry and causing widespread deployment failures.

---

## What Happened  
- **Registry authentication failure**:  
  - Registry migrated from Docker Hub to private Harbor instance  
  - Image pull secrets referenced old registry URL (`registry.old.corp:5000`)  
  - Docker config JSON contained expired credentials  
- **Observed symptoms**:  
  - Pods stuck in `ImagePullBackOff` or `ErrImagePull` states  
  - Events showed `Failed to pull image: unauthorized: authentication required`  
  - Cluster-wide impact as multiple teams used same base images  
- **Root discovery**:  
  - Secrets lacked `imagePullSecrets` reference in pod spec  
  - Registry credentials expired during maintenance window  
  - Network policies blocked access to new registry endpoint  

---

## Diagnosis Steps  

### 1. Check pod status and events:
```sh
kubectl get pods -A | grep -E "ImagePullBackOff|ErrImagePull"
kubectl describe pod <failing-pod> | grep -A10 Events
# Output: "Failed to pull image: unauthorized: authentication required"
```

### 2. Verify image pull secrets:
```sh
kubectl get secret -A -o yaml | grep -A5 "dockerconfigjson"
# Showed base64 encoded credentials for old registry
```

### 3. Test registry access:
```sh
kubectl run -it --rm registry-test --image=alpine -- \
  docker login registry.new.corp:5000 -u $USER -p $PASS
# Error: "Error response from daemon: Get https://registry.new.corp:5000/v2/: x509: certificate signed by unknown authority"
```

### 4. Check pod image pull spec:
```sh
kubectl get pod <pod> -o jsonpath='{.spec.containers[].image}'
# Output: registry.new.corp:5000/app:v1.2.3
```

---

## Root Cause  
**Authentication and configuration mismatch**:  
1. Image pull secrets referenced wrong registry endpoint  
2. Expired credentials not rotated  
3. Missing `imagePullSecrets` in pod/service account specs  
4. Self-signed registry certificate not trusted  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Create updated image pull secret
kubectl create secret docker-registry harbor-credentials \
  --docker-server=registry.new.corp:5000 \
  --docker-username=robot\$deploy \
  --docker-password=$(aws secretsmanager get-secret-value --secret-id registry-creds --query SecretString --output text) \
  --docker-email=devops@corp.com \
  --namespace=production

# 2. Patch deployments with correct secret
kubectl patch deployment myapp -n production -p '{
  "spec": {
    "template": {
      "spec": {
        "imagePullSecrets": [{"name": "harbor-credentials"}]
      }
    }
  }
}'

# 3. Force pod recreation
kubectl rollout restart deployment -n production
```

### Long-term Solution:
```yaml
# ServiceAccount with automounted image pull secrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-sa
  namespace: production
imagePullSecrets:
- name: harbor-credentials
---
# Pod using the service account
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: deploy-sa  # Automatically uses attached secrets
      containers:
      - name: app
        image: registry.new.corp:5000/app:v1.2.3
```

---

## Lessons Learned  
⚠️ **Registry migrations are high-risk**: Requires coordinated updates across all clusters  
⚠️ **Credentials have lifecycle**: Must be rotated before expiration  
⚠️ **ServiceAccounts centralize secret management**: Better than per-pod secrets  

---

## Prevention Framework  

### 1. Automated Secret Rotation
```yaml
# External Secrets Operator configuration
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: registry-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: harbor-credentials
    creationPolicy: Owner
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: registry/creds
      property: dockerconfig
```

### 2. Pre-flight Validation
```sh
# Image pull validation in CI/CD pipeline
validate_image_pull() {
  local image=$1
  local secret=$2
  
  # Test pull with secret
  kubectl run -it --rm pull-test --image=$image \
    --overrides="$(cat <<EOF
{
  "spec": {
    "imagePullSecrets": [{"name": "$secret"}],
    "containers": [{
      "name": "test",
      "image": "$image",
      "command": ["sleep", "infinity"]
    }]
  }
}
EOF
)" -- sleep 5 || {
    echo "ERROR: Failed to pull image $image"
    exit 1
  }
}
```

### 3. Monitoring & Alerting
```yaml
# Prometheus alerts for image pull issues
- alert: ImagePullFailures
  expr: increase(kubelet_image_pull_failures_total[5m]) > 5
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Image pull failures detected ({{ $value }} in 5m)"

- alert: RegistryCertificateExpiry
  expr: probe_ssl_earliest_cert_expiry{job="registry-tls-check"} - time() < 86400 * 7
  for: 5m
  labels:
    severity: warning
```

### 4. Registry Health Checks
```yaml
# Liveness probe for registry connectivity
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-check
data:
  check.sh: |