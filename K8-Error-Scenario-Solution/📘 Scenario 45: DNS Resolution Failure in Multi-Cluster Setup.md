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
