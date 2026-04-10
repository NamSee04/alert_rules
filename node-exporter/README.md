# Node Exporter Alert Rules

Production-ready Prometheus alert rules for comprehensive host-level monitoring using [Node Exporter](https://github.com/prometheus/node_exporter).

## Overview

This alert rule set provides **high-signal, low-noise** monitoring focused on actionable incidents that require human intervention. The rules have been curated to eliminate alert fatigue while catching critical infrastructure issues before they impact production.

**Design Principles:**
- **Actionable only**: Every alert requires a human response
- **Context-aware delays**: Filters transient spikes with appropriate `for` durations
- **Predictive where possible**: Catch issues before they become incidents
- **Multi-tier severity**: Critical issues vs warnings requiring investigation

## File

| File | Description |
|------|-------------|
| `node-exporter.yml` | 13 curated Prometheus alert rules for host metrics |

---

## Alert Rules Detail

### 🔴 Host Availability

#### `HostDown`
**Severity:** Critical  
**Fires when:** Node Exporter unreachable for 5 minutes  
**PromQL:** `up{job="node"} == 0`

**What it means:**
- The host is down or unreachable
- Node Exporter process has crashed
- Network connectivity lost to the host
- Prometheus cannot scrape the target

**Response:**
1. Check host availability via ping/SSH
2. Verify Node Exporter service status
3. Check network connectivity
4. Investigate recent changes or deployments
5. Check hardware status/console logs

**Tuning:**
- Adjust `job="node"` to match your Prometheus scrape config job name
- Increase delay beyond 5m if you have flaky networks

---

### 💾 Memory Alerts

#### `HostOutOfMemory`
**Severity:** Warning  
**Fires when:** Available memory drops below 20% for 2 minutes  
**PromQL:** `(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < .20)`

**What it means:**
- System is running low on available memory
- May lead to OOM kills if memory continues to fill
- Performance degradation likely (swapping, disk cache pressure)

**Response:**
1. Identify top memory consumers: `ps aux --sort=-%mem | head -20`
2. Check for memory leaks in applications
3. Review recent deployments
4. Consider horizontal scaling or memory upgrade
5. Check if caches can be cleared safely

**Tuning:**
- Lower threshold to 10% for critical tier: `< .10`
- Increase threshold to 30% for more conservative alerting
- Adjust delay for environment-specific patterns

**Related metrics:**
- `node_memory_MemTotal_bytes`: Total system memory
- `node_memory_MemAvailable_bytes`: Available memory (includes reclaimable caches)
- `node_memory_Cached_bytes`: Page cache size

---

### 🌐 Network Alerts

#### `HostUnusualNetworkThroughputIn`
**Severity:** Warning  
**Fires when:** Inbound bandwidth exceeds 80% of interface speed for 5 minutes  
**PromQL:** `((rate(node_network_receive_bytes_total[5m]) / on(instance, device) node_network_speed_bytes) > .80)`

**What it means:**
- Network interface is saturated with incoming traffic
- May indicate DDoS, traffic spike, or misconfigured load balancing
- Packet loss possible at sustained high utilization

**Response:**
1. Identify traffic sources via netstat/ss
2. Check if spike is legitimate (traffic surge, legitimate load)
3. Investigate potential DDoS or attack vectors
4. Review load balancer distribution
5. Consider scaling network capacity or rate limiting

**Tuning:**
- Remove alert entirely if bandwidth saturation is expected/normal
- Increase threshold to 90% for less sensitive alerting
- May fire on interfaces without proper speed detection (set `node_network_speed_bytes` manually)

#### `HostUnusualNetworkThroughputOut`
**Severity:** Warning  
**Fires when:** Outbound bandwidth exceeds 80% of interface speed for 5 minutes  
**PromQL:** `((rate(node_network_transmit_bytes_total[5m]) / on(instance, device) node_network_speed_bytes) > .80)`

**What it means:**
- Network interface saturated with outgoing traffic
- Possible data exfiltration, backup saturation, or legitimate surge

**Response:**
- Same as `HostUnusualNetworkThroughputIn`

**Note:** Both network alerts include a 5-minute delay to filter normal burst traffic.

---

### 💿 Disk Space Alerts

#### `HostOutOfDiskSpace`
**Severity:** Critical  
**Fires when:** Disk usage exceeds 90% (less than 10% available) for 2 minutes  
**PromQL:** `(node_filesystem_avail_bytes{fstype!~"^(fuse.*|tmpfs|cifs|nfs)"} / node_filesystem_size_bytes < .10 and on (instance, device, mountpoint) node_filesystem_readonly == 0)`

**What it means:**
- Filesystem is critically full
- Applications may fail to write logs, data, or temp files
- System instability imminent

**Response:**
1. Identify large files/directories: `du -hsx /* | sort -rh | head -10`
2. Check for runaway logs: `find /var/log -type f -size +100M`
3. Clear temporary files, caches, old logs
4. Archive or delete unnecessary data
5. Expand disk if structural issue
6. Investigate unexpected growth (runaway processes, log flooding)

**Filters:**
- Excludes `tmpfs`, `fuse`, `cifs`, `nfs` (usually not actionable)
- Only fires on writeable filesystems (readonly=0)

**Configuration requirement:**
```bash
node_exporter --collector.filesystem.ignored-mount-points='^/(sys|proc|dev|run)($|/)'
```

#### `HostDiskMayFillIn24Hours`
**Severity:** Warning  
**Fires when:** Linear prediction shows disk filling within 24 hours  
**PromQL:** `predict_linear(node_filesystem_avail_bytes{fstype!~"^(fuse.*|tmpfs|cifs|nfs)"}[6h], 86400) <= 0 and node_filesystem_avail_bytes > 0`

**What it means:**
- **Predictive alert**: Based on 6-hour trend, disk will be full in 24h
- Gives advance warning before `HostOutOfDiskSpace` fires
- Allows proactive intervention

**Response:**
1. Same investigation as `HostOutOfDiskSpace` but less urgent
2. Schedule cleanup or expansion before critical threshold
3. Identify growth trends: `df -h && df -h` (repeated over time)
4. Plan capacity expansion if structural

**Tuning:**
- Increase lookback window from `[6h]` to `[12h]` or `[24h]` for more stable predictions
- May fire false positives if workload patterns are cyclical (batch jobs)
- Reduce to `[3h]` for faster-changing environments

**Note:** Uses linear prediction - not suitable for exponential growth patterns.

#### `HostOutOfInodes`
**Severity:** Critical  
**Fires when:** Inode usage exceeds 90% (less than 10% available) for 2 minutes  
**PromQL:** `(node_filesystem_files_free / node_filesystem_files < .10 and on (instance, device, mountpoint) node_filesystem_readonly == 0)`

**What it means:**
- Filesystem running out of inodes (file metadata slots)
- Cannot create new files even if disk space available
- Common with millions of small files (caches, temp files, maildir)

**Response:**
1. Check inode usage: `df -i`
2. Find directories with excessive files: `for i in /*; do echo $i; find $i -type f | wc -l; done`
3. Common culprits: `/tmp`, `/var/spool`, email directories, package caches
4. Delete unnecessary small files
5. Consider filesystem with more inodes or restructure storage

**Troubleshooting:**
```bash
# Find directories with most files
du -a / | cut -d/ -f2 | sort | uniq -c | sort -nr | head
# Count inodes per directory
for d in /*; do echo "$d: $(find $d -type f | wc -l)"; done
```

#### `HostFilesystemDeviceError`
**Severity:** Critical  
**Fires when:** Filesystem device reports errors for 2 minutes  
**PromQL:** `node_filesystem_device_error{fstype!~"^(fuse.*|tmpfs|cifs|nfs)"} == 1`

**What it means:**
- Hardware failure (disk, RAID controller, SAN)
- Mount point issues
- Filesystem corruption
- I/O errors preventing filesystem stat operations

**Response:**
1. Check system logs immediately: `dmesg | tail -50`, `/var/log/syslog`
2. Check SMART status: `smartctl -a /dev/sdX`
3. Verify RAID status if applicable
4. Check mount points: `mount | grep error`
5. **Prepare for data loss/recovery** - may require immediate action
6. Consider failover to replica if available

**Urgency:** HIGH - indicates potential hardware failure or corruption

---

### ⚙️ CPU Alerts

#### `HostHighCpuLoad`
**Severity:** Warning  
**Fires when:** CPU utilization exceeds 80% for 10 minutes  
**PromQL:** `1 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) > .80`

**What it means:**
- Sustained high CPU usage across all cores
- System may be performance-degraded
- Possible runaway process or legitimate load spike

**Response:**
1. Identify CPU consumers: `top` or `htop`
2. Check for runaway processes
3. Investigate legitimate load increase (traffic surge?)
4. Consider horizontal scaling
5. Profile application for optimization opportunities
6. Check for cryptomining if unexplained

**Tuning:**
- 10-minute delay filters normal bursts (deployments, batch jobs)
- Increase threshold to 90% for less sensitive alerting
- Lower threshold to 70% for more conservative monitoring
- Adjust delay to 5m for faster response, 15m for more filtering

**Related metrics:**
- `node_cpu_seconds_total{mode="system"}`: Kernel CPU time
- `node_cpu_seconds_total{mode="user"}`: User process CPU time
- `node_cpu_seconds_total{mode="iowait"}`: CPU time waiting on I/O

---

### 🔧 System Alerts

#### `HostSystemdServiceCrashed`
**Severity:** Warning  
**Fires when:** Any systemd service enters `failed` state  
**PromQL:** `(node_systemd_unit_state{state="failed"} == 1)`

**What it means:**
- A systemd-managed service has crashed or failed to start
- May indicate application bug, configuration issue, or dependency failure

**Response:**
1. Identify failed service: `systemctl --failed`
2. Check service logs: `journalctl -u <service-name> -n 100`
3. Attempt restart: `systemctl restart <service-name>`
4. Investigate root cause in logs
5. Check recent configuration changes

**Tuning - IMPORTANT:**
This alert fires on **ALL** failed services. In production, you may want to filter to critical services only:

```yaml
expr: 'node_systemd_unit_state{state="failed",name=~"(sshd|docker|kubelet|nginx|postgresql|your-app).*"} == 1'
```

**Alternative:** Handle via Alertmanager routing to suppress non-critical service failures.

#### `HostOomKillDetected`
**Severity:** Warning  
**Fires when:** Out-of-memory (OOM) kill events detected  
**PromQL:** `(increase(node_vmstat_oom_kill[1m]) > 0)`

**What it means:**
- Linux kernel killed a process due to memory exhaustion
- System was critically low on memory
- Random processes may have been terminated

**Response:**
1. Check dmesg logs: `dmesg | grep -i "killed process"`
2. Identify killed process and investigate why
3. Check for memory leaks
4. Review memory limits (cgroups, containers)
5. Consider increasing system memory
6. May be preceded by `HostOutOfMemory` alert

**Related:**
- OOM kills are last resort when system cannot free memory
- Process selection is semi-random (oom_score based)
- Critical services may be killed

**Tuning:**
- 1-minute delay prevents single-spike false positives
- Increase to 2m for more filtering

#### `HostClockSkew`
**Severity:** Warning  
**Fires when:** System clock offset exceeds 50ms for 5 minutes  
**PromQL:** `(node_timex_offset_seconds > 0.05 or node_timex_offset_seconds < -0.05)`

**What it means:**
- System clock is drifting from NTP source
- Time synchronization issues

**Why this matters:**
- TLS certificates may fail validation
- Distributed systems rely on synchronized clocks (databases, Kubernetes, logs)
- Log correlation becomes impossible
- Authentication tokens may expire prematurely or be rejected

**Response:**
1. Check NTP/chrony status: `timedatectl status` or `chronyc tracking`
2. Verify NTP servers are reachable
3. Force time sync: `sudo chronyc makestep` or `sudo ntpd -gq`
4. Check for hardware issues (CMOS battery)
5. Verify timezone settings

**Tuning:**
- 50ms threshold appropriate for most systems
- Reduce to 10ms for very time-sensitive applications
- Increase to 100ms for less critical systems

#### `HostConntrackTableFull`
**Severity:** Critical  
**Fires when:** Netfilter connection tracking table exceeds 80% for 5 minutes  
**PromQL:** `(node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8)`

**What it means:**
- Linux connection tracking table is almost full
- **When full, new connections are silently dropped** (no error to client)
- Common with NAT, Docker, Kubernetes, or high-connection workloads

**Response:**
1. Check current usage: `cat /proc/sys/net/netfilter/nf_conntrack_count` vs `cat /proc/sys/net/netfilter/nf_conntrack_max`
2. Increase limit temporarily: `echo 262144 > /proc/sys/net/netfilter/nf_conntrack_max`
3. Make permanent: Add to `/etc/sysctl.conf`: `net.netfilter.nf_conntrack_max=262144`
4. Investigate connection leaks: `conntrack -L | wc -l`
5. Check for connection exhaustion attacks
6. Review application connection pooling
7. Consider reducing timeout values

**Common causes:**
- Kubernetes services with many endpoints  
- Docker with iptables backend
- NAT gateways handling many connections
- Connection leaks in applications
- DDoS or connection flood attacks

**Tuning:**
- Default Linux limit: 65536 connections
- Recommended for busy systems: 262144 or higher
- Each tracked connection uses ~300 bytes of kernel memory

---

## Prerequisites

### Required

- **Prometheus** (v2.0+) with recording rules and alerting enabled
- **Node Exporter** (v1.0+) running on all monitored hosts
  - Default port: `9100`
  - Systemd collector enabled: `--collector.systemd`
  - Filesystem collector configured (see below)

### Recommended Node Exporter Configuration

```bash
# Standard deployment
./node_exporter \
  --collector.filesystem.ignored-mount-points='^/(sys|proc|dev|run)($|/)' \
  --collector.filesystem.ignored-fs-types='^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$' \
  --collector.systemd
```

### Prometheus Configuration

Add to `prometheus.yml`:

```yaml
# Scrape config
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: 
          - 'host1:9100'
          - 'host2:9100'
    scrape_interval: 15s
    scrape_timeout: 10s

# Rule files
rule_files:
  - '/etc/prometheus/rules/node-exporter.yml'
```

---

## Installation

1. **Copy the rule file:**
   ```bash
   sudo cp node-exporter.yml /etc/prometheus/rules/
   sudo chown prometheus:prometheus /etc/prometheus/rules/node-exporter.yml
   ```

2. **Validate syntax:**
   ```bash
   promtool check rules /etc/prometheus/rules/node-exporter.yml
   ```

3. **Reload Prometheus:**
   ```bash
   curl -X POST http://localhost:9090/-/reload
   # OR
   sudo systemctl reload prometheus
   ```

4. **Verify rules loaded:**
   Navigate to Prometheus UI → Status → Rules
   - Confirm "NodeExporter" group appears
   - Verify 13 rules loaded

---

## Testing & Validation

### Test Alert Firing

Trigger a test alert to verify Alertmanager pipeline:

```bash
# Test disk space alert (create large file)
dd if=/dev/zero of=/tmp/bigfile bs=1G count=10

# Test memory alert (consume memory)
stress --vm 1 --vm-bytes 90% --timeout 300s

# Test CPU alert
stress --cpu 8 --timeout 600s
```

### Query Alert States

Check active alerts in Prometheus:

```promql
ALERTS{alertname="HostOutOfMemory"}
```

Check pending alerts:

```promql
ALERTS{alertname="HostOutOfMemory",alertstate="pending"}
```

---

## Tuning Recommendations

### For Low-Traffic Environments

- Increase CPU threshold: 80% → 90%
- Remove network throughput alerts if not applicable
- Increase disk space warning: 10% → 5%

### For High-Traffic Production

- Add critical tier for disk < 5%
- Lower memory threshold: 20% → 30%
- Add `HostDown` duplicate with 1m delay for faster detection
- Filter `HostSystemdServiceCrashed` to critical services only

### For Cost Optimization

Remove these if not critical:
- Network throughput alerts (if bandwidth not constrained)
- Clock skew (if time sync not critical)
- Conntrack (if not using NAT/Docker/Kubernetes extensively)

---

## Alertmanager Integration

Example Alertmanager routing for these alerts:

```yaml
route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  
  routes:
    # Critical alerts to PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: false
    
    # Warnings to Slack
    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'slack'
    slack_configs:
      - api_url: '<webhook>'
        channel: '#alerts'
        
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '<key>'
```

---

## Troubleshooting

### Alerts Not Firing

1. **Check rule evaluation:**
   ```bash
   curl http://localhost:9090/api/v1/rules | jq .
   ```

2. **Query the metric directly:**
   ```promql
   up{job="node"}
   node_memory_MemAvailable_bytes
   ```

3. **Check Prometheus logs:**
   ```bash
   journalctl -u prometheus -f
   ```

### False Positives

- Review `for` duration - may need increase
- Check if threshold is appropriate for your environment
- Use Alertmanager silences for known maintenance windows
- Consider adding exclude labels

### Missing Metrics

If metrics not available:
- Verify Node Exporter version (need v1.0+)
- Check Node Exporter flags (systemd collector, etc.)
- Verify scrape succeeding: `up{job="node"} == 1`

---

## Maintenance

### Version History

- **v2.0** (2026-04-10): Reduced from 18 to 13 alerts, removed noisy rules, added HostDown/ClockSkew/ConntrackTableFull
- **v1.0** (2025): Initial comprehensive rule set

### Periodic Review

**Quarterly:**
- Review alert frequency and false positive rate
- Tune thresholds based on operational experience
- Archive/remove unused alerts

**After incidents:**
- Evaluate whether alerts fired appropriately
- Adjust delays or thresholds if needed
- Add new alerts for uncovered scenarios

---

## Reference

### Key Metrics

| Metric | Description | Unit |
|--------|-------------|------|
| `up` | Target scrape success | 0/1 |
| `node_memory_MemAvailable_bytes` | Available memory | bytes |
| `node_memory_MemTotal_bytes` | Total memory | bytes |
| `node_filesystem_avail_bytes` | Available disk space | bytes |
| `node_filesystem_size_bytes` | Total disk space | bytes |
| `node_filesystem_files_free` | Available inodes | count |
| `node_cpu_seconds_total` | CPU time per mode | seconds |
| `node_network_receive_bytes_total` | Network RX bytes | bytes |
| `node_network_transmit_bytes_total` | Network TX bytes | bytes |
| `node_vmstat_oom_kill` | OOM kill counter | count |
| `node_timex_offset_seconds` | Clock offset from NTP | seconds |
| `node_nf_conntrack_entries` | Active conntrack entries | count |

### Further Reading

- [Node Exporter Documentation](https://github.com/prometheus/node_exporter)
- [Prometheus Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)
- [Google SRE Book - Practical Alerting](https://sre.google/sre-book/practical-alerting/)

---

## License

MIT License - modify and adapt to your environment as needed.
