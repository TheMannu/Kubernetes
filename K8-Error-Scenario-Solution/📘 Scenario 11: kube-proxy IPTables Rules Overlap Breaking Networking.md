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

### Long-term Solution:
1. **Isolate custom rules**:
   ```sh
   # Create dedicated chain
   iptables -t nat -N CUSTOM-NAT
   iptables -t nat -A PREROUTING -j CUSTOM-NAT
   ```

### Long-term Solution:
1. **Isolate custom rules**:
   ```sh
   # Create dedicated chain
   iptables -t nat -N CUSTOM-NAT
   iptables -t nat -A PREROUTING -j CUSTOM-NAT
   ```

2. **kube-proxy configuration**:
   ```yaml
   # /var/lib/kube-proxy/configuration.conf
   iptables:
     minSyncPeriod: 5s  # Faster recovery from conflicts
     localhostNodePorts: false  # Reduce rule complexity
   ```

---

## Lessons Learned  
⚠️ **kube-proxy owns NAT chains**: Never manually modify `KUBE-*` chains  
⚠️ **Rule ordering matters**: First-match wins in IPTables  
⚠️ **Silent failures**: kube-proxy retries indefinitely without alerting  

---

## Prevention Framework  

### 1. Firewall Policy
```sh
# Safe rule addition template
iptables -t nat -N MY-APP-NAT
iptables -t nat -A PREROUTING -j MY-APP-NAT  # Runs BEFORE KUBE-SERVICES
iptables -t nat -A POSTROUTING -j MY-APP-NAT
```

### 2. Admission Control
```yaml
# OPA Gatekeeper policy to block node modifications
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedSysctls
metadata:
  name: deny-iptables-mod
spec:
  match:
    kinds:
    - Node
  parameters:
    forbiddenPatterns: ["*iptables*"]
```

### 3. Monitoring
```yaml
# Prometheus alerts for kube-proxy health
- alert: KubeProxyRuleSyncFailed
  expr: increase(kube_proxy_sync_proxy_rules_errors_total[5m]) > 3
  labels:
    severity: critical
```

### 4. Documentation Standard
```yaml
## Node Firewall Policy
- All custom rules MUST:
  - Use dedicated chains (`-N CUSTOM-NAME`)
  - Never modify `KUBE-*` chains directly  
  - Be version-controlled in `/etc/iptables.rules`
```

---

**Key Metrics to Monitor**:  
- `kube_proxy_sync_proxy_rules_duration_seconds`  
- `node_netstat_IpExt_OutNoRoutes`  
- `kube_proxy_network_programming_duration_seconds`  

**Debugging Tools**:  
```sh
# Compare expected vs actual rules
diff <(kube-proxy --cleanup) <(iptables-save)

# Rule visualization
iptables -t nat -L -v --line-numbers
```

**kube-proxy Health Checks**:  
```sh
# Verify rule sync status
curl -s http://localhost:10249/proxyMode
# Should return "iptables" with no errors
``` 

**Backup Procedure**:  
```sh
# Daily iptables backup
iptables-save > /etc/iptables.rules.$(date +%F)
```
---