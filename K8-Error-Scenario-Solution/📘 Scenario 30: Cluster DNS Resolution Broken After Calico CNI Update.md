# 📘 Scenario 30: Cluster DNS Resolution Broken After Calico CNI Update

**Category**: Cluster Networking  
**Environment**: Kubernetes 1.23, Self-managed Calico 3.22  
**Impact**: Cluster-wide DNS outage lasting 53 minutes  

---

## Scenario Summary  
A Calico CNI upgrade introduced default-deny network policies that inadvertently blocked CoreDNS egress traffic, breaking all DNS resolution across the cluster.

---

## What Happened  
- **Calico upgrade**:  
  - v3.21 → v3.22 included new `default-deny` global network policy  
  - Changed default behavior from `allow-all` to explicit whitelisting  
- **Immediate symptoms**:  
  - `kubectl exec` commands hung with `i/o timeout`  
  - CoreDNS logs showed `SERVFAIL` for all queries  
  - `tcpdump` revealed `ICMP admin prohibited` packets  
- **Policy analysis**:  
  - New `default-deny` policy applied to `kube-system` namespace  
  - No exceptions for `kube-dns` service IP range  

---

## Diagnosis Steps  

### 1. Verify DNS functionality:
```sh
kubectl run -it --rm dns-test --image=busybox -- nslookup kubernetes.default
# Timeout after 30s
```

### 2. Inspect CoreDNS logs:
```sh
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# Showed "SERVFAIL" errors
```

### 3. Check network policies:
```sh
kubectl get globalnetworkpolicies.crd.projectcalico.org -o yaml | grep -i deny
# Output: "default-deny: true"
```

### 4. Analyze traffic flow:
```sh
kubectl debug node/<node> -it --image=nicolaka/netshoot -- \
  tcpdump -i any -nn "host 10.96.0.10 and port 53"
# Showed ICMP type=3 code=13 (Admin Prohibited)
```

---