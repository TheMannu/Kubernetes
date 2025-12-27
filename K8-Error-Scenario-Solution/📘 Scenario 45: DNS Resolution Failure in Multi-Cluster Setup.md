# 📘 Scenario 45: DNS Resolution Failure in Multi-Cluster Setup

**Category**: Multi-Cluster Networking  
**Environment**: Kubernetes 1.23, Multi-Cluster Federation with KubeFed  
**Impact**: Cross-cluster service communication failures affecting distributed applications  

---

## Scenario Summary  
DNS resolution failures between federated clusters prevented services from discovering each other, breaking distributed applications that relied on cross-cluster communication.

---

## What Happened  
- **Federation setup**:  
  - Two clusters (us-west, eu-central) federated using KubeFed  
  - Services exported via `FederatedService` resources  
- **DNS failures**:  
  - Queries to `service-name.namespace.svc.clusterset.local` timed out  
  - CoreDNS logs showed `NXDOMAIN` for federated service records  
  - Application logs showed `Name or service not known` errors  
- **Impact**:  
  - Global load balancing failed  
  - Active-active database replication broken  
  - Cross-region microservice communication disrupted  

---

## Diagnosis Steps  

### 1. Test DNS resolution:
```sh
kubectl run -it --rm dns-test --image=busybox -- \
  nslookup frontend.default.svc.clusterset.local
# Output: `server can't find frontend.default.svc.clusterset.local: NXDOMAIN`
```

### 2. Check CoreDNS configuration:
```sh
kubectl -n kube-system get configmap coredns -o yaml | yq '.data.Corefile'
# Missing `clusterset.local` zone configuration
```

### 3. Verify federated service status:
```sh
kubectl get federatedservice -A
# Showed services but no DNS endpoints
```

### 4. Inspect CoreDNS logs:
```sh
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 | grep -i clusterset
# No queries logged for clusterset.local domain
```

---

## Root Cause  
**DNS zone configuration gap**:  
1. CoreDNS not configured for federated domain (`clusterset.local`)  
2. Missing `kubefed` DNS provider integration  
3. No DNS record propagation between clusters  

---

## Fix/Workaround  

### Immediate Resolution:
```yaml
# Update CoreDNS configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        # Add federated domain
        federation clusterset.local {
           upstream
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### Long-term Solution:
```yaml
# KubeFed configuration with DNS integration
apiVersion: core.kubefed.io/v1beta1
kind: KubeFedConfig
metadata:
  name: kubefed-config
  namespace: kube-federation-system
spec:
  dns:
    domain: clusterset.local
    backend: coredns
  featureGates:
    PushReconciler: true
    SchedulerPreferences: true
```

---


## Lessons Learned  
⚠️ **Federation requires DNS coordination**: Multiple DNS zones must be synchronized  
⚠️ **Cross-cluster DNS is complex**: Different from single-cluster service discovery  
⚠️ **Silent failures**: Applications fail without clear DNS error messages  

---

## Prevention Framework  

### 1. DNS Validation Tooling
```sh
# Multi-cluster DNS health check script
check_federated_dns() {
  for cluster in us-west eu-central; do
    kubectl config use-context $cluster
    kubectl run -it --rm dns-check --image=busybox --restart=Never -- \
      nslookup test-service.default.svc.clusterset.local || {
        echo "ERROR: DNS resolution failed in $cluster"
        exit 1
      }
  done
}
```

### 2. Configuration Management
```yaml
# Helm values for CoreDNS with federation support
coredns:
  customCorefile: |
    federation clusterset.local {
       upstream
    }
  service:
    clusterIP: 10.96.0.10
  isClusterService: true
```

### 3. Monitoring
```yaml
# Prometheus alerts for federated DNS
- alert: FederatedDNSFailure
  expr: increase(coredns_dns_responses_total{zone="clusterset.local",rcode="NXDOMAIN"}[5m]) > 10
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Federated DNS resolution failing ({{ $value }} NXDOMAIN/min)"

- alert: CrossClusterDNSLatency
  expr: histogram_quantile(0.95, rate(coredns_dns_request_duration_seconds_bucket{zone="clusterset.local"}[5m])) > 1
  for: 5m
  labels:
    severity: warning
```

### 4. CI/CD Validation
```sh
# Pre-deployment DNS validation
validate_federated_dns() {
  local service=$1
  for cluster in $(kubectl config get-contexts -o name | grep -v '*' | grep -v kind); do
    if ! kubectl --context $cluster exec -it dns-test -- \
      nslookup $service.default.svc.clusterset.local >/dev/null 2>&1; then
      echo "ERROR: Service $service not resolvable in cluster $cluster"
      exit 1
    fi
  done
}
```

---

**Key DNS Components for Federation**:  
- **CoreDNS Federation Plugin**: Cross-cluster DNS resolution  
- **ExternalDNS**: Cloud provider DNS integration  
- **KubeFed DNS Controller**: Service endpoint propagation  
- **Global Load Balancer**: External access to federated services  

**Debugging Tools**:  
```sh
# Query CoreDNS directly
kubectl run -it --rm dig --image=infoblox/dnstools -- \
  dig @10.96.0.10 frontend.default.svc.clusterset.local +short
