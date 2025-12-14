# 📘 Scenario 41: Cluster Upgrade Failing Due to CNI Compatibility

**Category**: Cluster Networking & Upgrades  
**Environment**: Kubernetes 1.21 → 1.22, Custom CNI Plugin  
**Impact**: Complete pod networking failure post-upgrade, 3-hour outage  

---

## Scenario Summary  
A Kubernetes control plane upgrade to v1.22 broke all pod networking due to an incompatible version of the custom CNI plugin, rendering the cluster partially functional but unable to route pod-to-pod traffic.

---

## What Happened  
- **Control plane upgrade**:  
  - Successful `kubeadm upgrade` from 1.21.4 to 1.22.3  
  - Worker nodes remained on 1.21 temporarily  
- **Network symptoms**:  
  - Existing pods lost network connectivity (`No route to host`)  
  - New pods stuck in `ContainerCreating` with `CNI network not ready`  
  - CNI plugin DaemonSet logs showed `incompatible API version` errors  
- **Root discovery**:  
  - CNI plugin v0.9.0 only supported Kubernetes ≤1.21  
  - Required v1.0.0+ for 1.22 compatibility  

---

## Diagnosis Steps  

### 1. Verify pod networking:
```sh
kubectl run -it --rm net-test --image=nicolaka/netshoot -- ping 8.8.8.8
# Output: "connect: Network is unreachable"
```

### 2. Check CNI plugin status:
```sh
kubectl get pods -n kube-system -l app=cni-plugin
# Showed CrashLoopBackOff
```

### 3. Inspect CNI plugin logs:
```sh
kubectl logs -n kube-system -l app=cni-plugin --tail=50
# Output: "failed to create CNI network: incompatible API version"
```

### 4. Verify CNI configuration:
```sh
kubectl debug node/<node> -it --image=busybox -- cat /etc/cni/net.d/10-custom-cni.conf
# Showed CNI version "0.4.0" vs required "1.0.0"
```

---

### 3. Inspect CNI plugin logs:
```sh
kubectl logs -n kube-system -l app=cni-plugin --tail=50
# Output: "failed to create CNI network: incompatible API version"
```

### 4. Verify CNI configuration:
```sh
kubectl debug node/<node> -it --image=busybox -- cat /etc/cni/net.d/10-custom-cni.conf
# Showed CNI version "0.4.0" vs required "1.0.0"
```

---

## Fix/Workaround  

### Emergency Rollback:
```sh
# Option 1: Rollback control plane (if within rollback window)
kubeadm upgrade apply v1.21.4 --force

# Option 2: Upgrade CNI plugin immediately
kubectl apply -f https://raw.githubusercontent.com/cni-plugin/releases/v1.0.0/deploy.yaml
```

### Recovery Procedure:
```sh
# 1. Upgrade CNI DaemonSet
kubectl set image daemonset/cni-plugin -n kube-system cni-plugin=mycni/cni-plugin:v1.0.0

# 2. Restart kubelet on all nodes
kubectl get nodes -o name | xargs -I{} kubectl debug {} -it --image=busybox -- systemctl restart kubelet

# 3. Verify network recovery
kubectl run -it --rm connectivity-test --image=alpine -- ping -c 3 kubernetes.default.svc.cluster.local
```

---

## Lessons Learned  
⚠️ **CNI is a critical dependency**: Must be upgraded in lockstep with Kubernetes  
⚠️ **Version matrices matter**: Always check compatibility before upgrades  
⚠️ **Partial upgrades are dangerous**: Control plane and CNI must be compatible  

---

## Prevention Framework  

### 1. Upgrade Compatibility Checklist
```markdown
## Pre-Upgrade Validation
1. [ ] Check CNI plugin compatibility matrix  
2. [ ] Verify CNI configuration file version support  
3. [ ] Test upgrade in staging environment  
4. [ ] Prepare rollback procedure  
5. [ ] Document CNI upgrade steps separately  
```

### 2. Automated Validation
```sh
# Pre-upgrade compatibility check script
check_cni_compatibility() {
  local k8s_version=$1
  local cni_version=$(kubectl get ds cni-plugin -n kube-system -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d: -f2)
  
  case $k8s_version in
    1.22|1.23)
      if [[ $cni_version < v1.0.0 ]]; then
        echo "ERROR: CNI plugin $cni_version incompatible with Kubernetes $k8s_version"
        exit 1
      fi
      ;;
  esac
}
```

### 3. Monitoring
```yaml
# Prometheus alerts for CNI health
- alert: CNINotReady
  expr: kube_node_status_condition{condition="NetworkUnavailable",status="true"} == 1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "CNI networking unavailable on {{ $labels.node }}"

- alert: PodNetworkErrors
  expr: rate(kubelet_network_plugin_operations_errors_total[5m]) > 0
  for: 2m
  labels:
    severity: warning
```

### 4. Documentation Standards
```markdown
## CNI Upgrade Protocol
**Supported Version Matrix**:
| Kubernetes | CNI Plugin | Notes                    |
|------------|------------|--------------------------|
| 1.21.x     | v0.9.x     | Maximum support          |
| 1.22.x     | v1.0.x     | Breaking API changes     |
| 1.23.x     | v1.1.x     | Added new features       |

**Upgrade Order**:
1. Drain nodes
2. Upgrade CNI plugin
3. Upgrade kubelet
4. Uncordon nodes
```

---

**Key Compatibility Requirements**:  
- CNI spec version compatibility  
- Kubernetes API group changes  
- RBAC permission requirements  
- Container runtime interface changes  

**Debugging Tools**:  
```sh
# Check CNI plugin version
kubectl get ds -n kube-system -l app=cni-plugin -o yaml | grep image:

# Verify CNI binary presence
kubectl debug node/<node> -it --image=busybox -- ls -la /opt/cni/bin/

# Test CNI configuration
kubectl debug node/<node> -it --image=busybox -- cat /etc/cni/net.d/*.conf | jq .cniVersion
```