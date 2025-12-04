# 📘 Scenario 39: CRI Socket Mismatch Preventing kubelet Startup

**Category**: Container Runtime  
**Environment**: Kubernetes 1.22, Docker → containerd Migration  
**Impact**: Node registration failures during cluster-wide runtime migration  

---

## Scenario Summary  
A misconfigured CRI socket path prevented kubelet from communicating with containerd after a Docker-to-containerd migration, causing nodes to fail startup and become unavailable.

---