
# 📘 Scenario 54: Failed Node Recovery Due to Corrupt Kubelet Configuration

**Category**: Node Provisioning & Configuration Management  
**Environment**: Kubernetes 1.23, Bare Metal, Chef-managed nodes  
**Impact**: 24-hour node outage requiring manual intervention, capacity reduction  

---

## Scenario Summary  
A corrupted kubelet configuration file prevented a node from rejoining the cluster after maintenance, requiring manual recovery and extending downtime beyond the maintenance window.

---

## What Happened  
- **Post-maintenance node failure**:  
  - Node drained successfully for kernel upgrade  
  - Post-reboot, kubelet failed to start with `failed to load kubelet config file`  
  - Configuration file `/var/lib/kubelet/config.yaml` contained invalid YAML  
- **Diagnosis findings**:  
  - Journal logs showed `error unmarshaling JSON: while parsing JSON`  
  - File corruption due to incomplete Chef run during maintenance  
  - No configuration validation on write operations  
  - Backup configuration was 30 days old (outdated)  
- **Recovery challenges**:  
  - Node-specific certificates embedded in config  
  - Manual recovery required physical access (no out-of-band management)  

---

## Diagnosis Steps  

### 1. Check kubelet service status:
```sh
systemctl status kubelet --no-pager
# Output: "Failed to start kubelet: failed to load kubelet config file"
```

### 2. Inspect kubelet logs:
```sh
journalctl -u kubelet --no-pager -n 50 | grep -i "config\|json\|yaml"
# Output: "error unmarshaling JSON: while parsing JSON: unexpected end of JSON input"
```

### 3. Verify configuration file integrity:
```sh
# Check file size (corruption indicator)
ls -lh /var/lib/kubelet/config.yaml
# Output: 0 bytes (empty file)

# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('/var/lib/kubelet/config.yaml'))"
# Output: yaml.parser.ParserError: while parsing a block mapping
```

### 4. Check for backups:
```sh
find /etc/kubernetes /var/backups -name "*kubelet*config*" -type f 2>/dev/null
# Found outdated backup from 30 days ago
```

---

## Root Cause  
**Configuration management failure**:  
1. Chef run interrupted during config file write  
2. No file integrity validation post-write  
3. Missing configuration versioning or rollback mechanism  
4. Inadequate backup frequency for node-specific configs  

---

## Fix/Workaround  

### Emergency Recovery:
```sh
# 1. Restore from last known good configuration
cp /etc/kubernetes/kubelet.conf.$(date +%Y%m%d) /var/lib/kubelet/config.yaml

# 2. If no backup, rebuild configuration
cat > /var/lib/kubelet/config.yaml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
serializeImagePulls: false
clusterDomain: cluster.local
clusterDNS:
  - 10.96.0.10
resolvConf: /etc/resolv.conf
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
EOF

# 3. Rejoin cluster
kubeadm join 10.0.0.1:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> \
  --node-name $(hostname) --v=5
```

### Long-term Solution:
```yaml
# Configuration management with validation
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubelet-base-config
  namespace: kube-system
data:
  kubelet-base.yaml: |
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    # Base configuration here
---
