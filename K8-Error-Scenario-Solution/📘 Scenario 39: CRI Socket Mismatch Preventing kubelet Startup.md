# 📘 Scenario 39: CRI Socket Mismatch Preventing kubelet Startup

**Category**: Container Runtime  
**Environment**: Kubernetes 1.22, Docker → containerd Migration  
**Impact**: Node registration failures during cluster-wide runtime migration  

---

## Scenario Summary  
A misconfigured CRI socket path prevented kubelet from communicating with containerd after a Docker-to-containerd migration, causing nodes to fail startup and become unavailable.

---

## What Happened  
- **Runtime migration**:  
  - Cluster-wide initiative to migrate from Docker to containerd  
  - containerd installed but kubelet configuration not updated  
- **Startup failures**:  
  - `kubelet` failed with `failed to connect to CRI socket: socket /run/dockershim.sock not found`  
  - Nodes stuck in `NotReady` state with `ContainerRuntimeNotReady` condition  
- **Inconsistent state**:  
  - Some nodes migrated successfully, others failed  
  - Mixed runtime environment created operational complexity  

---

## Diagnosis Steps  

### 1. Check kubelet status:
```sh
systemctl status kubelet --no-pager
# Output: "Failed to start kubelet: failed to connect to CRI socket"
```

### 2. Inspect kubelet logs:
```sh
journalctl -u kubelet --no-pager -n 100 | grep -i "CRI\|socket"
# Showed: "connect: no such file or directory: /run/dockershim.sock"
```

### 3. Verify CRI socket existence:
```sh
ls -la /run/containerd/containerd.sock
# Socket existed but kubelet wasn't configured to use it
```

### 4. Check kubelet configuration:
```sh
cat /var/lib/kubelet/kubeadm-flags.env
# Output: --container-runtime=remote --container-runtime-endpoint=unix:///run/dockershim.sock
```

---

## Root Cause  
**Configuration drift during migration**:  
1. containerd installed but kubelet not reconfigured  
2. No validation step post-installation  
3. Inconsistent migration procedures across nodes  

---

## Fix/Workaround  

### Immediate Resolution:
```sh
# 1. Update kubelet configuration
echo 'KUBELET_KUBEADM_ARGS="--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock"' \
  > /var/lib/kubelet/kubeadm-flags.env

# 2. Reload systemd and restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 3. Verify node registration
kubectl get nodes <node-name>
```

### Automated Migration Script:
```sh
#!/bin/bash
# migrate-cri.sh
set -e

echo "Stopping kubelet..."
systemctl stop kubelet

echo "Updating CRI socket configuration..."
sed -i 's|dockershim\.sock|containerd/containerd\.sock|g' /var/lib/kubelet/kubeadm-flags.env

echo "Restarting kubelet..."
systemctl daemon-reload
systemctl start kubelet

echo "Verifying node status..."
sleep 10
kubectl get nodes $(hostname) -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

---

## Lessons Learned  
⚠️ **CRI configuration is critical**: kubelet cannot function without valid CRI socket  
⚠️ **Migration requires coordination**: All configuration files must be updated  
⚠️ **Validation is mandatory**: Must verify runtime connectivity post-changes  

---

## Prevention Framework  

### 1. Migration Automation
```yaml
# Ansible playbook for CRI migration
- name: Migrate from Docker to containerd
  hosts: k8s_nodes
  tasks:
  - name: Install containerd
    apt:
      name: containerd.io
      state: latest
  
  - name: Update kubelet configuration
    lineinfile:
      path: /var/lib/kubelet/kubeadm-flags.env
      regexp: 'dockershim\.sock'
      line: '--container-runtime-endpoint=unix:///run/containerd/containerd.sock'
    
  - name: Restart kubelet
    systemd:
      name: kubelet
      state: restarted
      
  - name: Verify node health
    command: kubectl get node {{ inventory_hostname }}
    register: node_status
    until: node_status.stdout | search("Ready")
    retries: 10
    delay: 30
```

### 2. Pre-flight Validation
```sh
# Pre-migration validation script
validate_cri_migration() {
  # Check containerd installation
  if ! systemctl is-active containerd > /dev/null; then
    echo "ERROR: containerd not running"
    exit 1
  fi
  
  # Verify socket exists
  if [ ! -S /run/containerd/containerd.sock ]; then
    echo "ERROR: containerd socket not found"
    exit 1
  fi
  
  # Test CRI connectivity
  if ! crictl version > /dev/null 2>&1; then
    echo "ERROR: Cannot connect to containerd via CRI"
    exit 1
  fi
}
```

### 3. Monitoring
```yaml
# Prometheus alerts for CRI issues
- alert: KubeletCRIConnectionFailed
  expr: kubelet_runtime_operations_errors_total > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Kubelet CRI connection failing ({{ $value }} errors)"

- alert: ContainerRuntimeNotReady
  expr: kube_node_status_condition{condition="ContainerRuntimeNotReady",status="true"} == 1
  for: 5m
  labels:
    severity: warning
```

### 4. Documentation Standards
```markdown
## CRI Migration Checklist
1. **Pre-migration**:
   - [ ] Backup kubelet configuration
   - [ ] Install and test containerd
   - [ ] Drain node (`kubectl drain`)
   
2. **Migration**:
   - [ ] Stop kubelet
   - [ ] Update `/var/lib/kubelet/kubeadm-flags.env`
   - [ ] Start kubelet
   
3. **Post-migration**:
   - [ ] Verify node status (`kubectl get node`)
   - [ ] Test pod scheduling
   - [ ] Remove Docker packages
```

---

**Key Configuration Files**:  
- `/var/lib/kubelet/kubeadm-flags.env` - kubelet runtime configuration  
- `/etc/containerd/config.toml` - containerd configuration  
- `/etc/crictl.yaml` - CRI CLI configuration  
