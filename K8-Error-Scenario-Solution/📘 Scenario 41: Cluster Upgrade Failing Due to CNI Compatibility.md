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
