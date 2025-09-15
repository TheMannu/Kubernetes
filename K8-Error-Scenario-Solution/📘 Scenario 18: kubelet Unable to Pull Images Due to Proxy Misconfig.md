# 📘 Scenario 18: kubelet Unable to Pull Images Due to Proxy Misconfig

**Category**: Cluster Networking  
**Environment**: Kubernetes v1.25, Corporate Proxy Environment  
**Impact**: Deployment failures across 200+ nodes, 4-hour service disruption  

---

## Scenario Summary  
A missing `NO_PROXY` configuration caused kubelet to route all container image pulls through an external proxy, breaking internal registry access and cluster DNS resolution.

---

## What Happened  
- **Proxy enforcement rollout**:  
  - Corporate policy mandated HTTP_PROXY for all outbound traffic  
  - Kubelet config updated with `HTTP_PROXY=http://proxy.corp:3128`  
- **Failure symptoms**:  
  - New pods stuck in `ImagePullBackOff`  
  - `kubelet` logs showed `Failed to pull image: context deadline exceeded`  
  - Internal registry metrics showed 100% connection timeouts  
- **Network analysis**:  
  - Internal `10.0.100.0/24` traffic was being routed externally  
  - CoreDNS queries to `kubernetes.default.svc` timed out  

---

## Diagnosis Steps  

### 1. Verify image pull failures:
```sh
kubectl get pods -A -o wide | grep -E 'ImagePullBackOff|ErrImagePull'
```

### 2. Inspect kubelet proxy config:
```sh
systemctl show kubelet --property=Environment --no-pager
# Output showed HTTP_PROXY set but NO_PROXY missing
```

### 3. Test cluster DNS through proxy:
```sh
kubectl run -it --rm netcheck --image=busybox -- \
  sh -c "wget -O- http://kubernetes.default.svc --proxy=on"
# Failed with 504 Gateway Timeout
```

### 4. Check registry access:
```sh
kubectl debug node/<node> -it --image=alpine -- \
  curl -x $HTTP_PROXY https://registry.internal.corp/v2/
# Returned 407 Proxy Authentication Required
```

---

## Root Cause  
**Proxy overreach**:  
1. Missing `NO_PROXY` exclusions for:  
   - Cluster CIDR (`10.0.0.0/8`)  
   - Service CIDR (`192.168.0.0/16`)  
   - Internal domains (`*.svc.cluster.local`)  
2. Proxy server blocked internal IP ranges  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Edit kubelet service config
sudo mkdir -p /etc/systemd/system/kubelet.service.d
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.corp:3128"
Environment="HTTPS_PROXY=http://proxy.corp:3128"
Environment="NO_PROXY=10.0.0.0/8,192.168.0.0/16,.svc,.cluster.local,localhost,127.0.0.1"
EOF

# 2. Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Cluster-wide Update:
```yaml
# Ansible playbook snippet
- name: Configure kubelet proxy
  template:
    src: proxy.conf.j2
    dest: /etc/systemd/system/kubelet.service.d/proxy.conf
    owner: root
    group: root
  vars:
    no_proxy_ranges:
      - "10.0.0.0/8"
      - "{{ kube_service_addresses }}"
      - ".{{ cluster_domain }}"
```

---

## Lessons Learned  
⚠️ **Proxies break cluster networking**: Must exclude all internal traffic  
⚠️ **Kubelet has special requirements**: Needs direct access to DNS + registry  
⚠️ **Corporate policies need adaptation**: Kubernetes isn't a standard workload  

---

## Prevention Framework  

### 1. Proxy Configuration Template
```ini
# /etc/systemd/system/kubelet.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.corp:3128"
Environment="HTTPS_PROXY=http://proxy.corp:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,.svc,.cluster.local,.internal.corp"
```

### 2. Validation Checks
```sh
# Pre-apply proxy test
validate_proxy() {
  kubectl run -it --rm proxy-test --image=alpine -- \
    sh -c "env | grep -E 'NO_PROXY|cluster.local' || exit 1"
}
```

### 3. Monitoring
```yaml
# Prometheus alerts
- alert: ProxyImagePullFailures
  expr: increase(kubelet_image_pull_failures_total[1h]) > 10
  labels:
    severity: critical
  annotations:
    summary: "Image pulls failing through proxy (instance {{ $labels.instance }})"
```

### 4. Documentation Standards

## Corporate Proxy Requirements for Kubernetes
- **Always exclude**:
  - Cluster CIDR (e.g., `10.0.0.0/8`)
  - Service CIDR (e.g., `192.168.0.0/16`)
  - DNS suffixes (`.svc`, `.cluster.local`)

- **Test before rollout**:
  ```sh
  kubectl run proxy-test --image=busybox -- \
    wget -O- http://kubernetes.default.svc
  ```

---

**Key NO_PROXY Entries for Kubernetes**:  
- Cluster Pod CIDR  
- Service CIDR  
- `.svc`, `.svc.cluster.local`  
- Internal registry domains  
- Node IP ranges 
