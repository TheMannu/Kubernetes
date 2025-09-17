# 📘 Scenario 19: Multiple Nodes Marked Unreachable Due to Flaky Network Interface

**Category**: Infrastructure Reliability  
**Environment**: Kubernetes v1.22, Bare-metal, Bonded NICs  
**Impact**: Intermittent node failures, pod evictions, and workload disruptions  

---

## Scenario Summary  
A flapping network interface caused multiple nodes to oscillate between `Ready` and `NotReady` states, triggering unnecessary pod evictions and workload rescheduling.

---

## What Happened  
- **Network instability**:  
  - Nodes randomly reported `NotReady` for 30-60 second intervals  
  - `kube-controller-manager` logs showed frequent `NodeNotReady` events
- **Observed symptoms**:  
  - Pods evicted with `NodeNetworkUnavailable` reason  
  - Cluster autoscaler provisioned unnecessary replacement nodes  
  - Storage systems (CSI) timed out during network drops  
- **Root discovery**:  
  - Switch logs showed `port up/down` events every 2-3 minutes  
  - `ethtool` reported `link flaps: 142` on affected nodes  

---

## Diagnosis Steps  

### 1. Check node status history:
```sh
kubectl get nodes -o wide --watch | tee node-status.log
# Showed flapping between Ready/NotReady
```

### 2. Inspect network interfaces:
```sh
ethtool eno1 | grep -A5 'Link detected'
# Reported intermittent link drops
```

### 3. Analyze kernel logs:
```sh
dmesg -T | grep -i 'link down\|nic'
# Revealed NIC resets
```

### 4. Verify switch port status:
```sh
ssh switch01 show interface ethernet 1/0/24 | include 'error|flap'
# Output: "Last link flapped: 00:02:15 ago"
```

---

## Root Cause  
**Physical layer instability**:  
1. **Faulty SFP+ transceiver** causing signal degradation  
2. **Loose fiber connection** in one port of the bonded interface  
3. **No LACP fallback** configured for single-port failures  

---

## Fix/Workaround  

### Immediate Actions:
```sh
# 1. Disable problematic switch port
ssh switch01 configure terminal
interface ethernet 1/0/24
shutdown

# 2. Drain affected nodes
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

### Permanent Solution:
```sh
# 1. Replace hardware (transceiver + cable)
# 2. Configure active-backup bonding:
nmcli con add type bond con-name bond0 ifname bond0 \
  mode active-backup miimon 100 primary eth0
nmcli con add type bond-slave ifname eth0 master bond0
nmcli con add type bond-slave ifname eth1 master bond0
```

---

## Lessons Learned  
⚠️ **Kubernetes amplifies physical issues**: Short blips cause pod evictions  
⚠️ **Bonding != redundancy**: Must test failover scenarios  
⚠️ **Hardware fails silently**: Requires active monitoring  

---

## Prevention Framework  

### 1. Network Bonding Configuration
```yaml
# NetworkManager bond example
connections:
- name: bond0
  type: bond
  interface-name: bond0
  bond:
    options:
      mode: "802.3ad"
      miimon: "100"
      lacp-rate: "fast"
```

### 2. Monitoring
```yaml
# Prometheus alerts
- alert: NodeNetworkFlapping
  expr: changes(kube_node_status_condition{condition="Ready",status="true"}[15m]) > 3
  for: 5m
  labels:
    severity: warning
    
- alert: NICErrorsHigh
  expr: node_network_up == 0 or rate(node_network_transmit_errs_total[5m]) > 5
  labels:
    severity: critical
```

### 3. Hardware Checks
```sh
# Daily NIC health check (via CronJob)
ethtool -S eth0 | grep -E 'err|drop'
smartctl -H /dev/nvme0
```

### 4. Documentation

## Bonding Best Practices
- **Mode**: 802.3ad (LACP) for switches, active-backup for redundancy  
- **Monitoring**: Track `carrier_changes` and `link_flaps`  
- **Testing**:  
  ```sh
  ip link set eth0 down && sleep 30 && ip link set eth0 up
  ```

---
