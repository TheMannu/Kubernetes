# 📘 Scenario #11: kube-proxy IPTables Rules Overlap Breaking Networking

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.22, On-prem, kube-proxy in IPTables mode  
**Impact**: Service connectivity loss, DNS resolution failures  

---

## Scenario Summary  
Custom IPTables NAT rules installed on nodes conflicted with kube-proxy's service routing rules, causing broken cluster networking and service blackholes.  

---

## What Happened  
- **Custom routing requirement**:  
  - Admin added NAT rules for external traffic redirection  
  - Rules modified `KUBE-SERVICES` chain directly  
- **Network symptoms**:  
  - Intermittent `Connection refused` for ClusterIP services  
  - CoreDNS resolution failures (`SERVFAIL`)  
  - NodePort services became unreachable  
- **kube-proxy behavior**:  
  - Continuous `Failed to ensure iptables rules` log warnings  
  - Rule ordering became corrupted after kube-proxy syncs  

---

## Diagnosis Steps  

### 1. Verify service connectivity:
```sh
kubectl run net-test --image=nicolaka/netshoot --rm -it -- \
   curl -v http://kubernetes.default.svc.cluster.local
```

### 2. Inspect IPTables rules:
```sh
iptables-save -t nat | grep -A 10 KUBE-SERVICES
# Found custom SNAT rules inserted above kube-proxy rules
```

### 3. Check kube-proxy logs:
```sh
journalctl -u kube-proxy --no-pager | grep -i conflict
# "Couldn't clean up iptables rules: exit status 2"
```

### 4. Compare rule versions:
```sh
# Before custom rules
iptables-save > /tmp/pre-change.iptables
# After incident
diff /tmp/pre-change.iptables <(iptables-save)
```

## Root Cause  
**Rule precedence conflict**:  
1. Custom rules inserted at **top** of `KUBE-SERVICES` chain  
2. kube-proxy rules **overwritten** during sync cycles (every 30s)  
3. No `-I` vs `-A` distinction in rule management  

---
## Fix/Workaround  

### Emergency Recovery:
```sh
# Flush ALL custom NAT rules (CAUTION: Verify first!)
iptables -t nat -F

# Restart kube-proxy to rebuild rules
systemctl restart kube-proxy

# Verify service restoration
kubectl get svc -A -o wide
```