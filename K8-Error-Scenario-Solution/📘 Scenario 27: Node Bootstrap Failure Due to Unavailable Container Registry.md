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


### Long-term Solution:
```yaml
# Packer template snippet for node AMI
provisioner "shell" {
  script = "preload-images.sh"
  environment_vars = [
    "KUBE_VERSION=v1.21.5",
    "REGISTRY=registry.internal:5000"
  ]
}
```
---


## Lessons Learned  
⚠️ **Bootstrap is fragile**: Requires all dependencies available  
⚠️ **Air-gapped needs redundancy**: Must have backup image sources  
⚠️ **Node images should be atomic**: Include all needed containers  

---

## Prevention Framework  

### 1. Image Preloading
```sh
# preload-images.sh
IMAGES=$(kubeadm config images list --kubernetes-version $KUBE_VERSION)
for image in $IMAGES; do
  docker pull $REGISTRY/${image#*/}
  docker save -o /opt/k8s/images/${image##*/}.tar $REGISTRY/${image#*/}
done
```

### 2. Registry Redundancy
```yaml
# containerd config.toml
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.internal"]
      endpoint = ["https://registry-01.internal", "https://registry-02.internal"]
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: RegistryUnavailable
  expr: probe_success{job="registry-healthcheck"} == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Container registry {{ $labels.instance }} is down"
```

### 4. Provisioning Checks
```yaml
# Ansible pre-task
- name: Verify registry access
  uri:
    url: "https://{{ registry_url }}/v2/"
    status_code: 200
    timeout: 5
  register: registry_check
  failed_when: false
  changed_when: false

- name: Fail if registry unreachable
  fail:
    msg: "Registry {{ registry_url }} is unavailable"
  when: registry_check.status != 200
```

---

**Key Images to Preload**:  
- `pause` (kubelet sandbox)  
- CNI plugins (`calico`, `flannel`)  
- `kube-proxy`  
- `node-exporter`  

**Debugging Tools**:  
```sh
# Check loaded images
ctr -n k8s.io images list

# Verify image pull secrets
kubectl get secret -n kube-system | grep registry

# Test registry access
kubectl run -it --rm registry-test --image=alpine -- \
  sh -c "apk add curl && curl -u user:pass https://registry.internal:5000/v2/_catalog"
```

**Bootstrap Hardening Checklist**:  
```markdown
1. [ ] Preload all control plane images in OS image  
2. [ ] Configure registry mirroring in containerd  
3. [ ] Document manual recovery procedures  
4. [ ] Test provisioning with registry offline  
```
---
---