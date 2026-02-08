
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

# DaemonSet to distribute and validate configs
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kubelet-config-manager
spec:
  template:
    spec:
      containers:
      - name: config-sync
        image: alpine/k8s:1.23.0
        command: ["/sync-config.sh"]
        volumeMounts:
        - name: kubelet-config
          mountPath: /etc/kubernetes/kubelet
```

---

## Lessons Learned  
⚠️ **Node configs are stateful**: Unique per node, can't be trivially replaced  
⚠️ **Atomic writes matter**: Partial config writes cause corruption  
⚠️ **Validation is crucial**: Must verify configs before service restart  

---

## Prevention Framework  

### 1. Configuration Management Automation
```sh
#!/bin/bash
# Safe kubelet config update script
update_kubelet_config() {
  local new_config=$1
  local backup_dir="/var/backups/kubelet/$(date +%Y%m%d)"
  
  # Validate new config
  if ! kubelet --validate --config="$new_config" --pod-manifest-path=/etc/kubernetes/manifests; then
    echo "ERROR: Configuration validation failed"
    exit 1
  fi
    
  # Create backup
  mkdir -p "$backup_dir"
  cp /var/lib/kubelet/config.yaml "$backup_dir/config.yaml.$(date +%s)"
  
  # Atomic write with rename
  cp "$new_config" "/var/lib/kubelet/config.yaml.new"
  mv "/var/lib/kubelet/config.yaml.new" "/var/lib/kubelet/config.yaml"
  
  # Restart with validation
  systemctl restart kubelet
  sleep 5
  systemctl is-active kubelet || {
    echo "ERROR: Kubelet failed to start, rolling back"
    cp "$backup_dir/config.yaml.$(date +%s)" /var/lib/kubelet/config.yaml
    systemctl restart kubelet
    exit 1
  }
}
```

### 2. Monitoring & Alerting
```yaml
# Prometheus alerts for kubelet configuration
- alert: KubeletConfigCorruption
  expr: kubelet_config_hash != on(node) kubelet_running_config_hash
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Kubelet configuration mismatch on {{ $labels.node }}"

- alert: KubeletConfigValidationFailed
  expr: increase(kubelet_config_validation_failures_total[5m]) > 0
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Kubelet configuration validation failures detected"

- alert: KubeletServiceDown
  expr: up{job="kubelet"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Kubelet down on {{ $labels.instance }}"
```

### 3. Configuration Backup Strategy
```yaml
# CronJob for regular configuration backups
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kubelet-config-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: backup
            image: alpine
            command:
            - /bin/sh
            - -c
            - |
              # Backup kubelet config
              NODE=$(hostname)
              BACKUP_DIR="/var/backups/kubelet/$(date +%Y%m%d)"
              mkdir -p $BACKUP_DIR
              cp /var/lib/kubelet/config.yaml $BACKUP_DIR/config.yaml.$NODE.$(date +%s)
              
              # Upload to S3 for disaster recovery
              aws s3 cp $BACKUP_DIR s3://my-cluster-backups/kubelet-configs/ --recursive
```

### 4. Validation Webhooks
```yaml
# Admission webhook for kubelet config validation
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: kubelet-config-validator
webhooks:
- name: kubelet-config.webhook.corp.com
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["configmaps"]
    scope: "Namespaced"
  failurePolicy: Fail
  sideEffects: None
  clientConfig:
    service:
      name: config-validator
      namespace: webhooks
      path: /validate/kubelet-config
```

---

**Key Configuration Components**:  
- **KubeletConfiguration**: Node-specific runtime configuration  
- **Bootstrap Config**: Initial join configuration  
- **Dynamic Kubelet Config**: Runtime configuration updates  
- **Certificate Files**: Client certificates for API server auth  

**Configuration Backup Strategy**:  
```markdown
## Backup Tiers
1. **Local (node)**: Hourly to `/var/backups/kubelet/`  
2. **Cluster-wide**: Daily to ConfigMap in `kube-system`  
3. **External**: Weekly to S3/GCS with 90-day retention  

## Critical Files to Backup
- `/var/lib/kubelet/config.yaml` (kubelet config)
- `/etc/kubernetes/kubelet.conf` (kubeconfig)
- `/var/lib/kubelet/pki/kubelet-client-current.pem` (certificate)
- `/etc/kubernetes/bootstrap-kubelet.conf` (bootstrap)
```

**Debugging Tools**:  
```sh
# Validate kubelet configuration
kubelet --validate --config=/var/lib/kubelet/config.yaml --pod-manifest-path=/etc/kubernetes/manifests

# Check configuration hash
sha256sum /var/lib/kubelet/config.yaml

# Compare running vs stored config
diff <(kubelet --print-config) /var/lib/kubelet/config.yaml

# Test configuration changes
kubelet --config=/tmp/test-config.yaml --pod-manifest-path=/tmp/manifests --exit-after-config

# Inspect kubelet bootstrap process
journalctl -u kubelet -f | grep -E "config\|certificate\|join"
```

**Recovery Procedures**:  
```markdown
## Automated Recovery Flow
1. **Detection**: Monitor for `kubelet_config_hash` mismatch  
2. **Rollback**: Restore from last known good backup  
3. **Validation**: Test config before restarting kubelet  
4. **Notification**: Alert administrators of recovery action  

## Manual Recovery Steps
1. SSH to affected node  
2. Stop kubelet: `systemctl stop kubelet`  
3. Restore config from backup  
4. Validate: `kubelet --validate --config=...`  
5. Start kubelet: `systemctl start kubelet`  
6. Verify node rejoins: `kubectl get nodes`  
```

**Bare Metal Specific Considerations**:  
```yaml
# PXE boot configuration with kubelet config
DEFAULT linux
LABEL linux
  KERNEL vmlinuz
  APPEND initrd=initrd.img ip=dhcp \
    kubelet.config.url=http://config-server/kubelet-config/{{.NodeName}}.yaml \
    kubelet.config.hash=$(sha256sum {{.NodeName}}.yaml) \
    kubelet.config.backup=http://config-server/backups/
```

**Chef/Ansible Best Practices**:  
```ruby
# Chef template with validation
template '/var/lib/kubelet/config.yaml' do
  source 'kubelet-config.yaml.erb'