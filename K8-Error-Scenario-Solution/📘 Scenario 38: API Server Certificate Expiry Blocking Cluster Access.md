# 📘 Scenario 38: API Server Certificate Expiry Blocking Cluster Access

**Category**: Control Plane Security  
**Environment**: Kubernetes 1.19, kubeadm  
**Impact**: Complete cluster inaccessibility for 2+ hours  

---

## Scenario Summary  
The API server's TLS certificate expired after 1 year, rendering the entire cluster inaccessible to all clients (`kubectl`, kubelets, controllers) and breaking all cluster operations.

---

## What Happened  
- **Certificate lifecycle failure**:  
  - kubeadm-generated certificates reached 1-year expiration  
  - No automatic renewal process in place  
  - Control plane operating for 395 days without certificate rotation  
- **Complete access loss**:  
  - `kubectl` commands failed with `x509: certificate has expired`  
  - Worker nodes lost connection to API server (`NodeNotReady`)  
  - Controller managers unable to sync resources  
- **Silent degradation**:  
  - No warnings until exact expiration moment  
  - All components failed simultaneously at expiration time  

---

## Diagnosis Steps  

### 1. Verify cluster accessibility:
```sh
kubectl get nodes
# Error: "Unable to connect to the server: x509: certificate has expired or is not yet valid"
```

### 2. Check certificate expiration:
```sh
# On control plane node
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
# Output: "notAfter=May 15 14:22:00 2023 GMT" (expired)
```

### 3. Inspect API server logs:
```sh
journalctl -u kube-apiserver --no-pager -n 50 | grep -i cert
# Showed TLS handshake failures
```

### 4. Verify all critical certificates:
```sh
kubeadm certs check-expiration
# Revealed multiple expired certificates
```

---

## Root Cause  
**Certificate management gap**:  
1. kubeadm default 1-year certificate lifespan  
2. No automated certificate renewal process  
3. Missing certificate expiration monitoring  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Renew all certificates (requires SSH access to control plane)
kubeadm certs renew all --config /etc/kubernetes/kubeadm-config.yaml

# 2. Restart control plane components
kubeadm init phase control-plane all --config /etc/kubernetes/kubeadm-config.yaml

# 3. Distribute new kubeconfigs
kubeadm init phase kubeconfig all --config /etc/kubernetes/kubeadm-config.yaml
cp /etc/kubernetes/admin.conf ~/.kube/config

# 4. Verify cluster recovery
kubectl get nodes
```

### Alternative Manual Recovery:
```sh
# If kubeadm unavailable, manually regenerate certificates
cd /etc/kubernetes/pki
rm -f apiserver.crt apiserver.key
kubeadm init phase certs apiserver --config /etc/kubernetes/kubeadm-config.yaml
systemctl restart kube-apiserver
```

---

## Lessons Learned  
⚠️ **Certificates have hard deadlines**: Expiration causes immediate failure  
⚠️ **kubeadm doesn't auto-renew**: Manual intervention required annually  
⚠️ **Silent time bombs**: No warnings until complete failure  

---

## Prevention Framework  

### 1. Automated Certificate Renewal
```yaml
# CronJob for certificate renewal
apiVersion: batch/v1
kind: CronJob
metadata:
  name: certificate-renewal
  namespace: kube-system
spec:
  schedule: "0 3 1 */6 *"  # Every 6 months at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: renew-certs
            image: k8s.gcr.io/kube-apiserver:v1.19.0
            command:
            - /bin/sh
            - -c
            - |
              kubeadm certs renew all
              systemctl restart kube-apiserver kube-controller-manager kube-scheduler
          restartPolicy: OnFailure
```

### 2. Certificate Monitoring
```yaml
# Prometheus alerts for certificate expiration
- alert: CertificateExpirySoon
  expr: kubelet_certificate_manager_client_ttl_seconds < 86400 * 30  # 30 days
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Kubernetes certificate expiring soon ({{ $value | humanizeDuration }})"

- alert: CertificateExpired
  expr: time() - kubelet_certificate_manager_client_expiration_seconds > 0
  for: 1m
  labels:
    severity: emergency
  annotations:
    summary: "Certificate has expired!"
```
