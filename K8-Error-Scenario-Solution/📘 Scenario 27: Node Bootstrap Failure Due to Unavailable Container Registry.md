# 📘 Scenario 27: Node Bootstrap Failure Due to Unavailable Container Registry

**Category**: Cluster Provisioning  
**Environment**: Kubernetes 1.21, On-prem, Air-gapped Registry  
**Impact**: 100% node provisioning failure during registry outage  

---

## Scenario Summary  
A private container registry outage prevented new nodes from joining the cluster by blocking pulls of critical bootstrap images (`pause`, `kube-proxy`, CNI), causing complete failure of scaling operations.

---

## What Happened  
- **Registry maintenance**:  
  - Storage backend upgrade caused 2-hour registry unavailability  
  - No maintenance window coordination with cluster operations  
- **Bootstrap failures**:  
  - `containerd` logged `failed to pull image: registry.internal:5000/pause:3.4.1`  
  - Nodes stuck in `NotReady` with `ContainerRuntimeNotReady` condition  
- **Cascading effects**:  
  - Cluster autoscaler triggered 15 failed node launches  
  - Emergency manual scaling attempts also failed  

---

## Diagnosis Steps  

### 1. Check node readiness:
```sh
kubectl get nodes -o json | \
  jq -r '.items[] | .metadata.name + " " + (.status.conditions[] | select(.type=="Ready") | .status + " " + .message'
```

### 2. Inspect container runtime:
```sh
journalctl -u containerd --no-pager -n 100 | grep -i pull
# Output: "PullImage registry.internal:5000/pause:3.4.1 failed"
```

### 3. Verify registry access:
```sh
curl -I https://registry.internal:5000/v2/
# Returned 503 Service Unavailable
```

### 4. Check essential images:
```sh
kubeadm config images list --kubernetes-version v1.21.5
# All images pointed to registry.internal
```

---

## Root Cause  
**Hard dependency on registry**:  
1. No local image cache for critical components  
2. No fallback mirror registry configured  
3. Infrastructure-as-Code (IaC) templates lacked image preloading  

---

## Fix/Workaround  

### Immediate Recovery:
```sh
# 1. Restore registry service
systemctl restart registry-backend

# 2. Manually load images on affected nodes
ctr -n k8s.io images import /opt/k8s/images/pause.tar
```

