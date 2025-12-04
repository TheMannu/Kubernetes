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
