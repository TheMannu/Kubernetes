# 📘 Scenario 10: Control Plane Unavailable After Flannel Misconfiguration  

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.18, On-prem (Bare Metal), Flannel CNI  
**Impact**: Complete control plane isolation, workload communication breakdown  

---

## Scenario Summary  
A new node joined with an incorrect pod CIDR, causing Flannel routing tables to corrupt and severing all control plane communications.  

---

## What Happened  
- **Node onboarding error**:  
  - New node configured with `10.244.1.0/24` when cluster expected `10.244.0.0/16`  
  - Flannel's `kube-subnet-mgr` failed to reconcile the mismatch  
- **Network symptoms**:  
  - API server became unreachable from worker nodes (`kubectl` timeout)  
  - Cross-node pod communication failed (`No route to host`)  
  - Flannel pods logged `"Failed to acquire lease"` errors  
- **Cascading failures**:  
  - Kube-proxy iptables rules became inconsistent  
  - Node status updates stopped flowing to control plane  

---

## Diagnosis Steps  

### 1. Verify cluster networking state:
```sh
kubectl get nodes -o custom-columns=NAME:.metadata.name,POD_CIDR:.spec.podCIDR
# Showed mismatched CIDRs across nodes

### 2. Check Flannel configuration:
```sh
kubectl -n kube-system get cm kube-flannel-cfg -o jsonpath='{.data.net-conf\.json}' | jq
# Revealed Network: "10.244.0.0/16" vs node's 10.244.1.0/24
```

### 3. Inspect network routes:
```sh
# On affected node:
ip route show | grep flannel
# Missing expected overlay routes
```

### 4. Analyze kubelet logs:
```sh
journalctl -u kubelet --no-pager | grep -i podcidr
# Log showed "PodCIDR not set for node" warnings
```

---

## Root Cause  
**CIDR misalignment**:  
1. **Manual node addition** bypassing cluster provisioning standards  
2. **No validation** of podCIDR consistency  
3. **Flannel's strict subnet requirements** not enforced  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# Drain and remove misconfigured node
kubectl drain <node> --delete-emptydir-data --force
kubectl delete node <node>

# Reset Flannel (on all nodes)
sudo systemctl restart flanneld
sudo iptables -F && sudo iptables -t nat -F

# Rejoin node with correct CIDR
kubeadm join ... --pod-network-cidr=10.244.0.0/16
```

### Long-term Solution:
1. **Automated node validation**:
   ```yaml
   # OPA Gatekeeper constraint
   apiVersion: constraints.gatekeeper.sh/v1beta1
   kind: K8sRequiredPodCIDR
   spec:
     match:
       kinds: [{kind: Node}]
     parameters:
       cidr: "10.244.0.0/16"
   ```

2. **Flannel configuration hardening**:
   ```json
   {
     "Network": "10.244.0.0/16",
     "Backend": {
       "Type": "vxlan",
       "DirectRouting": true
     }
   }
   ```

---

## Lessons Learned  
⚠️ **CNI plugins are fragile**: Flannel requires perfect CIDR alignment  
⚠️ **Manual node additions are dangerous**: Always use automation  
⚠️ **Control plane depends on pod network**: Broken CNI → Broken API  

---

## Prevention Framework  

### 1. Admission Control
```yaml
# Kyverno ClusterPolicy example
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-pod-cidr
spec:
  validationFailureAction: enforce
  rules:
  - name: check-pod-cidr
    match:
      resources:
        kinds:
        - Node
    validate:
      message: "Pod CIDR must match cluster range"
      pattern:
        spec:
          podCIDR: "10.244.0.0/16"
```

### 2. Provisioning Safeguards
```sh
# kubeadm pre-flight check
kubeadm join --config=kubeadm-config.yaml --dry-run
```

### 3. Monitoring
```yaml
# Critical Prometheus alerts
- alert: FlannelCIDRMismatch
  expr: count(kube_node_info{pod_cidr!="10.244.0.0/16"}) > 0
  labels:
    severity: critical
```
