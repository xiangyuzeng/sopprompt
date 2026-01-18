# Luckin Coffee DevOps Alert Investigation SOP
## EC2 Instance Diagnosis Edition v4.0
### Integrated with MCP Data Sources & System Architecture

---

## Document Info

| Field | Value |
|-------|-------|
| Version | 4.0 |
| Last Updated | 2025-01-11 |
| Scope | EC2 Instance Alerts (VM-level) |
| Related SOPs | RDS SOP (TBD), K8s SOP (TBD) |

---

## Available MCP Data Sources

Claude Code has access to **150+ data endpoints** via MCP:

| Data Source Type | Count | Primary Use |
|------------------|-------|-------------|
| Prometheus instances | 3 | Metrics, alerting |
| MySQL databases | 60 | Application data, logs |
| Redis clusters | 75 | Cache status, sessions |
| PostgreSQL databases | 3 | Dify, Map services |
| AWS Services | 7+ | CloudWatch, EKS, Cost |
| Grafana ecosystem | Full | Dashboards, alerts, incidents |

### Prometheus Datasources
| Datasource | UID | Use For |
|------------|-----|---------|
| **UMBQuerier-Luckin** | `df8o21agxtkw0d` | Primary metrics (default) - node metrics, app metrics |
| prometheus | `ff7hkeec6c9a8e` | General Prometheus metrics |
| prometheus_redis | `ff6p0gjt24phce` | Redis-specific metrics |

### Key MySQL Servers (by Category)
| Category | Servers |
|----------|---------|
| DevOps/Platform | aws-luckyus-devops-rw, aws-luckyus-ldas-rw, aws-luckyus-ldas01-rw |
| Sales/CRM | aws-luckyus-salescrm-rw, aws-luckyus-salesmarketing-rw, aws-luckyus-salesorder-rw |
| SCM | aws-luckyus-scm-shopstock-rw, aws-luckyus-scmcommodity-rw |
| Operations | aws-luckyus-opshop-rw, aws-luckyus-opproduction-rw |
| Finance | aws-luckyus-fitax-rw, aws-luckyus-ifiaccounting-rw |

### Key Redis Clusters (by Category)
| Category | Clusters |
|----------|----------|
| API/Gateway | luckyus-apigateway, luckyus-aapi-unionauth |
| Auth/Session | luckyus-auth, luckyus-session, luckyus-unionauth |
| Sales | luckyus-isales-crm, luckyus-isales-order, luckyus-isales-market |
| SCM | luckyus-scm-shopstock, luckyus-scm-commodity |
| Platform | luckyus-devops, luckyus-cmdb, luckyus-ldas |

### AWS Services Available
| Service | Capabilities |
|---------|-------------|
| CloudWatch Logs | Log groups, Log Insights queries |
| CloudWatch Metrics | EC2 metrics, custom metrics, alarms |
| CloudWatch Alarms | Active alarms, alarm history |
| EKS | Cluster management, pod logs, events |
| Cost Explorer | Cost analysis, forecasts |

---

## Lessons Learned (Incorporated)

| Incident | Root Cause | What Was Missed | Fix in SOP |
|----------|------------|-----------------|------------|
| 【vm-内存】P1 Jan 10, 2025 | Disk full (luckysddladmin logs) | Alert said memory, actual was disk | Phase 0: Alert validation |
| luckysddladmin disk alert | 40GB disk + no log rotation | Sibling at 88% about to fail | Phase 3: Sibling check |
| SkyWalking high memory | JVM heap pressure | Monitoring services have different patterns | Service-type checks |
| Data retention gap | Prometheus 15-90 days only | Couldn't do RCA for old incidents | Phase 1: Data availability |

---

## Quick Start Checklist

Before investigation:
- [ ] Alert name vs actual metric verified
- [ ] Timestamp converted to UTC
- [ ] Instance IP/hostname identified
- [ ] Service level determined (L0/L1/L2)
- [ ] Service owner identified
- [ ] Sibling instances listed
- [ ] Appropriate MCP data sources identified

---

## Service Level Priority Matrix

| Level | Definition | Response Time | Examples |
|-------|------------|---------------|----------|
| **L0** 🔴 | Core Business | <15 min | isalesorderservice, isalespaymentservice, iopshopservice |
| **L1** 🟡 | Important | <30 min | luckynacos, luckymdm, iupush*, API Gateway |
| **L2** 🟢 | Normal | <2 hours | skywalking, izeus*, luckysddladmin |

---

## Copy-Paste Investigation Prompt

```markdown
# EC2 Alert Investigation - Cross-System Diagnosis

## Alert Context
- **Alert Name:** [PASTE_FULL_ALERT_NAME]
- **Alert Metric:** [PASTE_METRIC]
- **Trigger Time:** [YYYY-MM-DD HH:MM TIMEZONE] (UTC: [XX:XX])
- **Duration:** [Duration or "ongoing"]
- **Status:** [Active / Resolved / Flapping]
- **Environment:** Luckin-PROD
- **Instance:** [IP or hostname]

---

## PHASE 0: Alert Validation ⚠️ CRITICAL FIRST STEP

**Alert names can be misleading! Verify the actual metric.**

| Alert Name Contains | Also Check | Common Mismatch |
|---------------------|------------|-----------------|
| 内存 (memory) | Memory AND Disk | Disk full can trigger memory alerts |
| 磁盘 (disk) | Disk AND I/O | High I/O can cause disk alerts |
| CPU | CPU AND Load | High load might be I/O wait |

### Query Active Alerts (Grafana MCP)
Use Grafana alerting to get current alert state:
- List all firing alerts for the instance
- Get alert rule details and actual metric expression
- Check alert history for pattern

### Verify Alert Metric (Prometheus MCP)
Using datasource: **UMBQuerier-Luckin** (UID: `df8o21agxtkw0d`)

```promql
# Get the actual alert status
ALERTS{alertname=~".*[ALERT_KEYWORD].*", instance=~".*[IP].*"}

# Check what triggered
ALERTS{alertstate="firing", instance=~".*[IP].*"}
```

---

## PHASE 1: Data Availability & Instance Identification

### 1.1 Identify EC2 Instance

**Prometheus Query** (UMBQuerier-Luckin):
```promql
# Find instance by IP
node_uname_info{instance=~".*[IP_ADDRESS].*"}

# Get all metadata
node_meta_info{instance=~".*[IP_ADDRESS].*"}

# List instances for a service
node_uname_info{nodename=~"[SERVICE_NAME].*"}
```

**AWS CloudWatch** (via MCP):
- Get EC2 instance details by IP
- Check instance status checks
- Get instance metadata (type, AZ, etc.)

Fill in:
| Field | Value |
|-------|-------|
| Instance IP | |
| Hostname | |
| Service Name | |
| Project | |
| Service Level | L0/L1/L2 |
| Service Owner | |
| DB Owner | |

### 1.2 Check Data Retention

**Prometheus** (UMBQuerier-Luckin):
```promql
# Check earliest data point
min_over_time(up{instance=~".*[IP].*"}[30d])
```

**MySQL - Alert Logs** (aws-luckyus-devops-rw):
```sql
-- Check alert log retention
SELECT MIN(create_time), MAX(create_time) 
FROM t_umb_alert_log 
WHERE instance LIKE '%[IP]%';

-- Verify data exists for incident time
SELECT COUNT(*) 
FROM t_umb_alert_log 
WHERE instance LIKE '%[IP]%'
  AND create_time BETWEEN '[INCIDENT_START]' AND '[INCIDENT_END]';
```

**AWS CloudWatch Logs** (via MCP):
- Query log groups for the service
- Check log retention settings
- Use Log Insights for timeframe

### 1.3 Identify ALL Sibling Instances ⚠️ CRITICAL

**Prometheus Query**:
```promql
# Find all siblings by naming pattern
node_uname_info{nodename=~"[SERVICE_PREFIX].*"}

# Example: For luckysddladmin01, find all luckysddladmin*
node_uname_info{nodename=~"luckysddladmin.*"}

# Get current status of all siblings
up{nodename=~"[SERVICE_PREFIX].*"}
```

Document all siblings:
| Sibling | IP | Status |
|---------|-----|--------|
| [name]01 | | Primary (alerting) |
| [name]02 | | Check immediately |
| [name]03 | | Check immediately |

---

## PHASE 2: Cross-System Health Check

**⚠️ MANDATORY: Run ALL checks regardless of alert type!**

### 2.1 Disk Analysis 🔴 ALWAYS CHECK FIRST

**Prometheus** (UMBQuerier-Luckin):
```promql
# Disk usage percentage (root partition)
(1 - node_filesystem_avail_bytes{instance=~".*[IP].*",mountpoint="/"} / node_filesystem_size_bytes{instance=~".*[IP].*",mountpoint="/"}) * 100

# ALL partitions
(1 - node_filesystem_avail_bytes{instance=~".*[IP].*",fstype!="tmpfs"} / node_filesystem_size_bytes) * 100

# 7-day trend
(1 - node_filesystem_avail_bytes{instance=~".*[IP].*",mountpoint="/"}[7d] / node_filesystem_size_bytes) * 100

# Growth rate (bytes per hour)
rate((node_filesystem_size_bytes{instance=~".*[IP].*",mountpoint="/"} - node_filesystem_avail_bytes{instance=~".*[IP].*",mountpoint="/"})[1h]) * 3600

# Fleet-wide: All instances above 80%
sort_desc((1 - node_filesystem_avail_bytes{env="Luckin-PROD",mountpoint="/"} / node_filesystem_size_bytes) * 100 > 80)

# Predict disk exhaustion (will hit 90% within 24h)
predict_linear(node_filesystem_avail_bytes{instance=~".*[IP].*",mountpoint="/"}[6h], 3600*24) / node_filesystem_size_bytes < 0.1

# Inode usage
(1 - node_filesystem_files_free{instance=~".*[IP].*",mountpoint="/"} / node_filesystem_files{instance=~".*[IP].*",mountpoint="/"}) * 100
```

**AWS CloudWatch Metrics** (via MCP):
```
# EC2 EBS metrics
DiskReadOps, DiskWriteOps
DiskReadBytes, DiskWriteBytes
VolumeQueueLength (for EBS)
```

**Shell Commands** (if SSH access available):
```bash
df -h
du -sh /* 2>/dev/null | sort -hr | head -20
du -sh /usr/local/springboot/logs/*/ 2>/dev/null | sort -hr
find / -xdev -type f -size +100M -exec ls -lh {} \; 2>/dev/null | head -10
df -i  # Inode check
```

### 2.2 Memory Analysis

**Prometheus** (UMBQuerier-Luckin):
```promql
# Memory usage (custom metric)
node_memory_app_usage_rate_service_avg{instance=~".*[IP].*"}

# Memory usage (calculated)
(1 - node_memory_MemAvailable_bytes{instance=~".*[IP].*"} / node_memory_MemTotal_bytes{instance=~".*[IP].*"}) * 100

# Top 10 memory consumers fleet-wide
topk(10, node_memory_app_usage_rate_service_avg{env="Luckin-PROD"})

# Memory trend (1 hour)
(1 - node_memory_MemAvailable_bytes{instance=~".*[IP].*"} / node_memory_MemTotal_bytes)[1h]

# OOM events
increase(node_vmstat_oom_kill{instance=~".*[IP].*"}[24h])

# Swap usage
node_memory_SwapTotal_bytes{instance=~".*[IP].*"} - node_memory_SwapFree_bytes{instance=~".*[IP].*"}
```

**AWS CloudWatch Metrics**:
```
MemoryUtilization (if CloudWatch agent installed)
SwapUtilization
```

### 2.3 CPU & Load Analysis

**Prometheus** (UMBQuerier-Luckin):
```promql
# CPU usage
100 - (avg by(instance) (rate(node_cpu_seconds_total{instance=~".*[IP].*",mode="idle"}[5m])) * 100)

# CPU by mode (check iowait!)
sum by(mode) (rate(node_cpu_seconds_total{instance=~".*[IP].*"}[5m])) * 100

# Load averages
node_load1{instance=~".*[IP].*"}
node_load5{instance=~".*[IP].*"}
node_load15{instance=~".*[IP].*"}

# Compare load to CPU count
node_load15{instance=~".*[IP].*"} / count without(cpu) (node_cpu_seconds_total{instance=~".*[IP].*",mode="idle"})
```

**AWS CloudWatch Metrics**:
```
CPUUtilization
CPUCreditUsage (for T-series instances)
CPUCreditBalance (for T-series instances)
```

### 2.4 Disk I/O Analysis

**Prometheus** (UMBQuerier-Luckin):
```promql
# I/O utilization
rate(node_disk_io_time_seconds_total{instance=~".*[IP].*"}[5m]) * 100

# Throughput
rate(node_disk_read_bytes_total{instance=~".*[IP].*"}[5m])
rate(node_disk_written_bytes_total{instance=~".*[IP].*"}[5m])

# IOPS
rate(node_disk_reads_completed_total{instance=~".*[IP].*"}[5m])
rate(node_disk_writes_completed_total{instance=~".*[IP].*"}[5m])

# I/O wait time
rate(node_disk_io_time_weighted_seconds_total{instance=~".*[IP].*"}[5m])
```

### 2.5 Network Analysis

**Prometheus** (UMBQuerier-Luckin):
```promql
# Traffic
rate(node_network_receive_bytes_total{instance=~".*[IP].*",device!="lo"}[5m])
rate(node_network_transmit_bytes_total{instance=~".*[IP].*",device!="lo"}[5m])

# Errors
rate(node_network_receive_errs_total{instance=~".*[IP].*"}[5m])
rate(node_network_transmit_errs_total{instance=~".*[IP].*"}[5m])

# TCP connections
node_netstat_Tcp_CurrEstab{instance=~".*[IP].*"}

# Connection states
node_sockstat_TCP_alloc{instance=~".*[IP].*"}
```

**AWS CloudWatch Metrics**:
```
NetworkIn, NetworkOut
NetworkPacketsIn, NetworkPacketsOut
```

---

## PHASE 3: Sibling Instance Comparison ⚠️ CRITICAL

**This phase found luckysddladmin02 at 88% BEFORE it alerted!**

### 3.1 Compare ALL Siblings

**Prometheus** (UMBQuerier-Luckin):
```promql
# Disk usage for all siblings
sort_desc((1 - node_filesystem_avail_bytes{nodename=~"[SERVICE].*",mountpoint="/"} / node_filesystem_size_bytes) * 100)

# Memory for all siblings
sort_desc(node_memory_app_usage_rate_service_avg{nodename=~"[SERVICE].*"})

# CPU for all siblings
sort_desc(100 - (avg by(instance,nodename) (rate(node_cpu_seconds_total{nodename=~"[SERVICE].*",mode="idle"}[5m])) * 100))

# Load for all siblings
sort_desc(node_load15{nodename=~"[SERVICE].*"})
```

### 3.2 Sibling Health Matrix

| Instance | IP | Disk % | Memory % | CPU % | Load | Risk | Hours to 90% |
|----------|-----|--------|----------|-------|------|------|--------------|
| [name]01 | | | | | | | |
| [name]02 | | | | | | | |
| [name]03 | | | | | | | |

**Risk Level:**
- 🔴 Critical: >90% any resource OR <2h to threshold
- ⚠️ Warning: >80% any resource OR <24h to threshold
- ✅ OK: All resources <80%

### 3.3 Time-to-Threshold Prediction

**Prometheus**:
```promql
# Hours until disk reaches 90% for each sibling
(node_filesystem_avail_bytes{nodename=~"[SERVICE].*",mountpoint="/"} - (node_filesystem_size_bytes * 0.1)) 
/ 
abs(deriv(node_filesystem_avail_bytes{nodename=~"[SERVICE].*",mountpoint="/"}[1h]))
/ 3600
```

**⚠️ Flag any sibling <24h to threshold for proactive action!**

---

## PHASE 4: Service-Type Specific Checks

### 4.1 Spring Boot Applications

**Prometheus** (UMBQuerier-Luckin):
```promql
# JVM heap usage (if exposed)
jvm_memory_used_bytes{instance=~".*[IP].*",area="heap"}
jvm_memory_max_bytes{instance=~".*[IP].*",area="heap"}

# GC activity
rate(jvm_gc_pause_seconds_sum{instance=~".*[IP].*"}[5m])
```

**MySQL** - Check related database (based on service):
```sql
-- Example: For Sales service, check aws-luckyus-salescrm-rw
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads_connected';

-- Slow queries
SELECT * FROM mysql.slow_log 
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY query_time DESC LIMIT 20;
```

**Redis** - Check related cache (based on service):
```
# Example: For Sales service, check luckyus-isales-crm
INFO memory
INFO clients
SLOWLOG GET 20
```

### 4.2 Monitoring Services (SkyWalking, Zeus)

**Prometheus**:
```promql
# These services normally run high memory
node_memory_app_usage_rate_service_avg{nodename=~"skywalking.*"}
node_memory_app_usage_rate_service_avg{nodename=~"izeus.*"}

# Check if above normal range (85% for monitoring services)
node_memory_app_usage_rate_service_avg{nodename=~"skywalking.*"} > 85
```

### 4.3 Nacos Cluster

**Prometheus**:
```promql
# Nacos cluster status (normally 77-80%)
node_memory_app_usage_rate_service_avg{nodename=~"luckynacos.*"}

# All Nacos instances
node_uname_info{nodename=~"luckynacos.*"}
```

**MySQL** (aws-luckyus-ldas-rw or nacos config DB):
```sql
-- Check Nacos config records
SELECT COUNT(*) FROM config_info;
SELECT COUNT(*) FROM his_config_info;
```

### 4.4 API Gateway Services

**Prometheus**:
```promql
# Connection count
node_netstat_Tcp_CurrEstab{nodename=~"luckyaapiproxy.*"}

# Network traffic
rate(node_network_receive_bytes_total{nodename=~"luckyaapiproxy.*",device!="lo"}[5m])
```

**Redis** (luckyus-apigateway):
```
INFO clients
CLIENT LIST | wc -l
```

### 4.5 DevOps/DBA Utilities (luckysddladmin, idbtask)

**Prometheus** (focus on disk):
```promql
# These services often have log accumulation
(1 - node_filesystem_avail_bytes{nodename=~"luckysddladmin.*",mountpoint="/"} / node_filesystem_size_bytes) * 100

# Growth rate
rate((node_filesystem_size_bytes{nodename=~"luckysddladmin.*",mountpoint="/"} - node_filesystem_avail_bytes)[1h]) * 3600
```

---

## PHASE 5: Database & Cache Correlation

### 5.1 MySQL Health Check

**If service connects to MySQL, check the related database server:**

| Service Category | MySQL Server |
|------------------|--------------|
| Sales/CRM | aws-luckyus-salescrm-rw, aws-luckyus-salesorder-rw |
| SCM | aws-luckyus-scm-shopstock-rw, aws-luckyus-scmcommodity-rw |
| Operations | aws-luckyus-opshop-rw, aws-luckyus-opproduction-rw |
| Finance | aws-luckyus-fitax-rw, aws-luckyus-ifiaccounting-rw |
| DevOps | aws-luckyus-devops-rw, aws-luckyus-ldas-rw |

**MySQL Queries**:
```sql
-- Connection status
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW PROCESSLIST;

-- Slow queries during incident
SELECT * FROM mysql.slow_log 
WHERE start_time BETWEEN '[INCIDENT_START]' AND '[INCIDENT_END]'
ORDER BY query_time DESC LIMIT 20;

-- Lock waits
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- Table sizes (if disk issue)
SELECT table_schema, table_name, 
       ROUND(data_length/1024/1024, 2) AS data_mb,
       ROUND(index_length/1024/1024, 2) AS index_mb
FROM information_schema.tables 
ORDER BY data_length DESC LIMIT 20;
```

### 5.2 Redis Health Check

**If service uses Redis, check the related cluster:**

| Service Category | Redis Cluster |
|------------------|---------------|
| Sales | luckyus-isales-crm, luckyus-isales-order |
| SCM | luckyus-scm-shopstock, luckyus-scm-commodity |
| Auth | luckyus-auth, luckyus-session |
| API Gateway | luckyus-apigateway |
| DevOps | luckyus-devops, luckyus-ldas |

**Redis Commands**:
```
# Memory status
INFO memory

# Client connections
INFO clients
CLIENT LIST

# Slow commands
SLOWLOG GET 20

# Key count and memory by db
INFO keyspace

# Big keys (can cause memory spikes)
MEMORY DOCTOR
```

**Prometheus** (prometheus_redis - UID: `ff6p0gjt24phce`):
```promql
# Redis memory usage
redis_memory_used_bytes{instance=~".*[REDIS_CLUSTER].*"}

# Connected clients
redis_connected_clients{instance=~".*[REDIS_CLUSTER].*"}

# Slow log length
redis_slowlog_length{instance=~".*[REDIS_CLUSTER].*"}
```

---

## PHASE 6: AWS CloudWatch Integration

### 6.1 EC2 Instance Metrics

**CloudWatch Metrics** (via MCP):
```
Namespace: AWS/EC2
Metrics:
  - CPUUtilization
  - NetworkIn, NetworkOut
  - DiskReadOps, DiskWriteOps
  - StatusCheckFailed_Instance
  - StatusCheckFailed_System
  
Dimensions:
  - InstanceId: [i-xxxxxxxxx]
```

### 6.2 CloudWatch Alarms

**Check related alarms**:
- List all alarms for the instance
- Get alarm history
- Check alarm state changes during incident

### 6.3 CloudWatch Logs

**Log Insights Query** (via MCP):
```
# Search for errors in application logs
fields @timestamp, @message
| filter @message like /ERROR|Exception|OOM/
| sort @timestamp desc
| limit 100

# Search for specific timeframe
fields @timestamp, @message
| filter @timestamp between [INCIDENT_START] and [INCIDENT_END]
| filter @message like /ERROR/
| stats count() by bin(5m)
```

---

## PHASE 7: Alert Correlation & History

### 7.1 Alert History from MySQL

**MySQL** (aws-luckyus-devops-rw):
```sql
-- Alerts on same instance (last 20)
SELECT 
    alert_name,
    alert_status,
    create_time,
    resolve_time,
    TIMESTAMPDIFF(MINUTE, create_time, COALESCE(resolve_time, NOW())) as duration_min
FROM t_umb_alert_log
WHERE instance LIKE '%[IP_ADDRESS]%'
ORDER BY create_time DESC
LIMIT 20;

-- Alerts in same timeframe (±30 min)
SELECT 
    alert_name,
    instance,
    alert_status,
    create_time
FROM t_umb_alert_log
WHERE create_time BETWEEN DATE_SUB('[INCIDENT_TIME]', INTERVAL 30 MINUTE) 
                      AND DATE_ADD('[INCIDENT_TIME]', INTERVAL 30 MINUTE)
ORDER BY create_time;

-- Recurring pattern for this service
SELECT 
    DATE(create_time) as alert_date,
    alert_name,
    COUNT(*) as occurrences
FROM t_umb_alert_log
WHERE instance LIKE '%[SERVICE_NAME]%'
GROUP BY DATE(create_time), alert_name
ORDER BY alert_date DESC
LIMIT 30;

-- Alert frequency by type
SELECT 
    alert_name,
    COUNT(*) as total_occurrences,
    COUNT(DISTINCT DATE(create_time)) as days_with_alerts,
    AVG(TIMESTAMPDIFF(MINUTE, create_time, resolve_time)) as avg_duration_min
FROM t_umb_alert_log
WHERE instance LIKE '%[SERVICE_NAME]%'
  AND create_time > DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY alert_name
ORDER BY total_occurrences DESC;
```

### 7.2 Grafana Alerting Integration

**Via Grafana MCP**:
- List all alert rules for the service
- Get alert rule history
- Check alert contact points for escalation
- Create incident if needed

---

## PHASE 8: Output Report Format

### Executive Summary
| Field | Value |
|-------|-------|
| **Root Cause** | [One-line summary] |
| **Affected Instance** | [hostname (IP)] |
| **Service Level** | [L0/L1/L2] |
| **Service Owner** | [Name] |
| **Impact** | [What was affected] |
| **Resolution** | [How resolved] |
| **Status** | ✅ Resolved / ⚠️ Monitoring / 🔴 Active |

### Alert Validation
| Item | Alert Said | Actually Found |
|------|------------|----------------|
| Resource Type | [e.g., Memory] | [e.g., Disk] |
| Metric | [Alert metric] | [Actual issue] |

### Data Sources Used
| Source | Type | Query/Check |
|--------|------|-------------|
| UMBQuerier-Luckin | Prometheus | Node metrics |
| [MySQL server] | MySQL | Alert logs |
| [Redis cluster] | Redis | Cache status |
| CloudWatch | AWS | EC2 metrics |

### Cross-System Check Results
| Resource | Status | Current | Threshold | Data Source | Findings |
|----------|--------|---------|-----------|-------------|----------|
| Disk | ✅/⚠️/🔴 | [X]% | 90% | Prometheus | |
| Memory | ✅/⚠️/🔴 | [X]% | 90% | Prometheus | |
| CPU | ✅/⚠️/🔴 | [X]% | 80% | Prometheus/CW | |
| Disk I/O | ✅/⚠️/🔴 | [X]% | 70% | Prometheus | |
| Network | ✅/⚠️/🔴 | [X] Mbps | - | Prometheus | |
| MySQL | ✅/⚠️/🔴 | [X] conn | - | MySQL | |
| Redis | ✅/⚠️/🔴 | [X] MB | - | Redis | |

### Sibling Instance Status
| Instance | IP | Disk % | Memory % | CPU % | Risk | Hours to Threshold |
|----------|-----|--------|----------|-------|------|-------------------|
| [name]01 | | | | | | |
| [name]02 | | | | | | |

**⚠️ Proactive Alert:** [Flag siblings approaching threshold]

### Related Database/Cache Status
| Resource | Server/Cluster | Status | Findings |
|----------|----------------|--------|----------|
| MySQL | [server-name] | ✅/⚠️/🔴 | |
| Redis | [cluster-name] | ✅/⚠️/🔴 | |

### Root Cause Analysis
[Detailed explanation]

### Resolution Steps
1. [Step taken]
2. [Step taken]

### Preventive Recommendations

**Priority 1 - Immediate**
| Action | Owner | Data Source to Monitor |
|--------|-------|----------------------|
| | | |

**Priority 2 - Short Term**
| Action | Owner | Data Source to Monitor |
|--------|-------|----------------------|

### Monitoring Improvements
- [ ] [New Prometheus alert rule]
- [ ] [New Grafana dashboard]
- [ ] [CloudWatch alarm]

---

## Appendix A: MCP Data Source Quick Reference

### Prometheus Datasources
| Name | UID | Use Case |
|------|-----|----------|
| UMBQuerier-Luckin | df8o21agxtkw0d | Primary - all node/app metrics |
| prometheus | ff7hkeec6c9a8e | General metrics |
| prometheus_redis | ff6p0gjt24phce | Redis-specific metrics |

### MySQL Servers by Service Area
| Area | Servers |
|------|---------|
| DevOps | aws-luckyus-devops-rw, aws-luckyus-ldas-rw |
| Sales | aws-luckyus-salescrm-rw, aws-luckyus-salesorder-rw, aws-luckyus-salesmarketing-rw |
| SCM | aws-luckyus-scm-shopstock-rw, aws-luckyus-scmcommodity-rw |
| Operations | aws-luckyus-opshop-rw, aws-luckyus-opproduction-rw |
| Finance | aws-luckyus-fitax-rw, aws-luckyus-ifiaccounting-rw |
| Auth | aws-luckyus-iluckyauthapi-rw, aws-luckyus-ipermission-rw |
| BigData | aws-luckyus-icyberdata-rw |

### Redis Clusters by Service Area
| Area | Clusters |
|------|----------|
| Gateway | luckyus-apigateway, luckyus-aapi-unionauth |
| Auth | luckyus-auth, luckyus-session, luckyus-unionauth |
| Sales | luckyus-isales-crm, luckyus-isales-order, luckyus-isales-market |
| SCM | luckyus-scm-shopstock, luckyus-scm-commodity |
| Platform | luckyus-devops, luckyus-cmdb, luckyus-ldas |

### PostgreSQL Servers
| Server | Service |
|--------|---------|
| aws-luckyus-dify-rw | Dify platform |
| aws-luckyus-difynew-rw | Dify (new) |
| aws-luckyus-pgilkmap-rw | Map service |

---

## Appendix B: Service Owner Quick Reference

| Team | Services | Primary Contact | DB Contact |
|------|----------|-----------------|------------|
| Sales | isales*, icdp* | 张晓松/张翔 | 钟志 |
| Finance | ifi*, iehr, iadmin | 陈亮/尤志杰 | 钟志 |
| SupplyChains | iscm* | 方思扬 | 钟志 |
| EEOP/Shop | iop* | 陈培浩 | 钟志 |
| MicroService | luckynacos, sentinel, gateway | 罗宁/翁延海 | 李经宇/张延伟 |
| Monitoring | skywalking, izeus*, umb* | 姚清居/郑小军 | 李培龙 |
| DevOps | luckysddladmin, luckyauth | 吴伟根 | 李培龙 |
| DBA | idbtask, idbcollect | 于伯伟/周争强 | 李经宇 |
| StoragePlatform | ldas*, luckyozono | 孔增/沈育斌 | 李经宇 |

---

## Appendix C: Key Prometheus Metrics

```yaml
# Disk
node_filesystem_avail_bytes
node_filesystem_size_bytes
node_filesystem_files_free

# Memory
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_memory_app_usage_rate_service_avg  # Custom Luckin metric

# CPU
node_cpu_seconds_total
node_load1, node_load5, node_load15

# Disk I/O
node_disk_io_time_seconds_total
node_disk_read_bytes_total
node_disk_written_bytes_total

# Network
node_network_receive_bytes_total
node_network_transmit_bytes_total
node_netstat_Tcp_CurrEstab

# Redis (prometheus_redis)
redis_memory_used_bytes
redis_connected_clients
redis_slowlog_length
```

---

## Appendix D: Investigation Completion Checklist

- [ ] **Phase 0:** Alert name vs actual metric validated
- [ ] **Phase 1:** Data availability confirmed, instance identified
- [ ] **Phase 2:** ALL resources checked (Disk FIRST, Memory, CPU, I/O, Network)
- [ ] **Phase 3:** ALL sibling instances checked
- [ ] **Phase 4:** Service-type specific checks completed
- [ ] **Phase 5:** Related MySQL/Redis checked
- [ ] **Phase 6:** AWS CloudWatch reviewed
- [ ] **Phase 7:** Alert history correlated
- [ ] **Phase 8:** Report generated with:
  - [ ] Root cause identified
  - [ ] Data sources documented
  - [ ] Sibling risks flagged
  - [ ] Owner notified
  - [ ] Recommendations provided

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-11 | Initial SOP |
| 2.0 | 2025-01-11 | EC2-specific, sibling checks |
| 3.0 | 2025-01-11 | Luckin architecture integration |
| 4.0 | 2025-01-11 | Full MCP data source integration (Prometheus, MySQL, Redis, PostgreSQL, AWS, Grafana) |
```
