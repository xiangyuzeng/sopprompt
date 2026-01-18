# EC2 Alert Investigation - Quick Prompt v4.0
## With MCP Data Source Integration
### Copy → Fill Placeholders → Run in Claude Code

---

```markdown
# EC2 Alert Investigation - Cross-System Diagnosis

## Alert Context
- **Alert Name:** [PASTE_ALERT_NAME]
- **Metric:** [PASTE_METRIC]
- **Time:** [YYYY-MM-DD HH:MM TZ] (UTC: [XX:XX])
- **Instance:** [IP or hostname]
- **Status:** [Active/Resolved]
- **Environment:** Luckin-PROD

---

## Phase 0: Alert Validation ⚠️ FIRST

**Alert names can be misleading! Verify actual metric.**

Use Grafana MCP to check actual alert state and metric expression.

| Alert Says | Also Check |
|------------|------------|
| Memory | Disk (logs filling disk) |
| Disk | I/O |
| CPU | Load, I/O wait |

---

## Phase 1: Instance & Data Check

**Prometheus** (UMBQuerier-Luckin - UID: df8o21agxtkw0d):
```promql
node_uname_info{instance=~".*[IP].*"}
```

**MySQL** (aws-luckyus-devops-rw) - Check data availability:
```sql
SELECT MIN(create_time), MAX(create_time) 
FROM t_umb_alert_log WHERE instance LIKE '%[IP]%';
```

Fill in: Instance IP, Hostname, Service, Level (L0/L1/L2), Owner

**List ALL siblings:**
```promql
node_uname_info{nodename=~"[SERVICE_PREFIX].*"}
```

---

## Phase 2: Cross-System Check (RUN ALL)

### 2.1 Disk ⚠️ CHECK FIRST
**Prometheus** (UMBQuerier-Luckin):
```promql
# Current usage
(1 - node_filesystem_avail_bytes{instance=~".*[IP].*",mountpoint="/"} / node_filesystem_size_bytes) * 100

# Fleet-wide >80%
sort_desc((1 - node_filesystem_avail_bytes{env="Luckin-PROD",mountpoint="/"} / node_filesystem_size_bytes) * 100 > 80)

# Predict exhaustion
predict_linear(node_filesystem_avail_bytes{instance=~".*[IP].*",mountpoint="/"}[6h], 3600*24) / node_filesystem_size_bytes < 0.1
```

### 2.2 Memory
**Prometheus**:
```promql
node_memory_app_usage_rate_service_avg{instance=~".*[IP].*"}
topk(10, node_memory_app_usage_rate_service_avg{env="Luckin-PROD"})
increase(node_vmstat_oom_kill{instance=~".*[IP].*"}[24h])
```

### 2.3 CPU & Load
**Prometheus**:
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{instance=~".*[IP].*",mode="idle"}[5m])) * 100)
node_load15{instance=~".*[IP].*"}
```

### 2.4 Disk I/O
```promql
rate(node_disk_io_time_seconds_total{instance=~".*[IP].*"}[5m]) * 100
```

### 2.5 Network
```promql
node_netstat_Tcp_CurrEstab{instance=~".*[IP].*"}
rate(node_network_receive_bytes_total{instance=~".*[IP].*",device!="lo"}[5m])
```

---

## Phase 3: Sibling Comparison ⚠️ CRITICAL

**Prometheus**:
```promql
# Compare all siblings
sort_desc((1 - node_filesystem_avail_bytes{nodename=~"[SERVICE].*",mountpoint="/"} / node_filesystem_size_bytes) * 100)
sort_desc(node_memory_app_usage_rate_service_avg{nodename=~"[SERVICE].*"})
```

| Instance | IP | Disk % | Memory % | CPU % | Risk | Hours to 90% |
|----------|-----|--------|----------|-------|------|--------------|
| | | | | | | |

**⚠️ Flag siblings >80% or <24h to threshold!**

---

## Phase 4: Service-Type Checks

**Spring Boot** - Check related MySQL/Redis:

| Service Area | MySQL Server | Redis Cluster |
|--------------|--------------|---------------|
| Sales | aws-luckyus-salescrm-rw | luckyus-isales-crm |
| SCM | aws-luckyus-scm-shopstock-rw | luckyus-scm-shopstock |
| Operations | aws-luckyus-opshop-rw | luckyus-shop |
| DevOps | aws-luckyus-devops-rw | luckyus-devops |

**MySQL Query**:
```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW PROCESSLIST;
```

**Redis Command**:
```
INFO memory
INFO clients
SLOWLOG GET 10
```

**Prometheus Redis** (prometheus_redis - UID: ff6p0gjt24phce):
```promql
redis_memory_used_bytes{instance=~".*[REDIS_CLUSTER].*"}
```

---

## Phase 5: Alert History

**MySQL** (aws-luckyus-devops-rw):
```sql
-- Same instance history
SELECT alert_name, alert_status, create_time, resolve_time
FROM t_umb_alert_log
WHERE instance LIKE '%[IP]%'
ORDER BY create_time DESC LIMIT 20;

-- Same timeframe (±30 min)
SELECT alert_name, instance, create_time
FROM t_umb_alert_log
WHERE create_time BETWEEN DATE_SUB('[TIME]', INTERVAL 30 MINUTE) 
                      AND DATE_ADD('[TIME]', INTERVAL 30 MINUTE)
ORDER BY create_time;
```

---

## Phase 6: AWS CloudWatch (via MCP)

Check:
- EC2 instance metrics (CPUUtilization, NetworkIn/Out)
- CloudWatch alarms for instance
- CloudWatch Logs for application errors

---

## Output Format Required

### Executive Summary
| Field | Value |
|-------|-------|
| Root Cause | |
| Instance | |
| Service Level | L0/L1/L2 |
| Owner | |
| Status | |

### Data Sources Used
| Source | Type | UID/Server |
|--------|------|------------|
| UMBQuerier-Luckin | Prometheus | df8o21agxtkw0d |
| [MySQL] | MySQL | |
| [Redis] | Redis | |

### Cross-System Results
| Resource | Status | Value | Source |
|----------|--------|-------|--------|
| Disk | ✅/⚠️/🔴 | % | Prometheus |
| Memory | ✅/⚠️/🔴 | % | Prometheus |
| CPU | ✅/⚠️/🔴 | % | Prometheus |
| MySQL | ✅/⚠️/🔴 | conn | MySQL |
| Redis | ✅/⚠️/🔴 | MB | Redis |

### Sibling Status
| Instance | Disk | Memory | Risk |
|----------|------|--------|------|
| | | | |

### Recommendations (with owner & priority)
```

---

## Quick Reference

### Prometheus Datasources
| Name | UID | Use |
|------|-----|-----|
| **UMBQuerier-Luckin** | df8o21agxtkw0d | Primary - node/app metrics |
| prometheus | ff7hkeec6c9a8e | General metrics |
| prometheus_redis | ff6p0gjt24phce | Redis metrics |

### MySQL by Service Area
| Area | Server |
|------|--------|
| DevOps/Alert Logs | aws-luckyus-devops-rw |
| Sales | aws-luckyus-salescrm-rw |
| SCM | aws-luckyus-scm-shopstock-rw |
| Operations | aws-luckyus-opshop-rw |
| Finance | aws-luckyus-fitax-rw |

### Redis by Service Area
| Area | Cluster |
|------|---------|
| Gateway | luckyus-apigateway |
| Auth | luckyus-auth, luckyus-session |
| Sales | luckyus-isales-crm, luckyus-isales-order |
| SCM | luckyus-scm-shopstock |
| Platform | luckyus-devops, luckyus-ldas |

### Service Level Response
| Level | Response | Examples |
|-------|----------|----------|
| L0 🔴 | <15 min | isalesorder*, iopshop*, iscmshopstock |
| L1 🟡 | <30 min | luckynacos, iupush*, gateway |
| L2 🟢 | <2 hours | skywalking, izeus*, luckysddladmin |

### Key Owners
| Area | Contact |
|------|---------|
| Sales | 张晓松, 张翔 |
| SCM | 方思扬 |
| Shop/EEOP | 陈培浩 |
| MicroService | 罗宁, 翁延海 |
| Monitoring | 姚清居 |
| DevOps | 吴伟根 |
| DBA | 钟志, 李经宇, 张延伟 |

---

## Investigation Checklist

### Pre-Flight
- [ ] Alert name vs actual metric verified
- [ ] UTC time calculated
- [ ] Instance identified
- [ ] Service level determined (L0/L1/L2)
- [ ] Sibling instances listed
- [ ] Relevant MySQL/Redis identified

### Post-Investigation
- [ ] ALL resources checked (Disk FIRST)
- [ ] ALL siblings checked
- [ ] Related MySQL/Redis checked
- [ ] AWS CloudWatch reviewed
- [ ] Alert history correlated
- [ ] Root cause identified
- [ ] Proactive risks flagged
- [ ] Owner notified
- [ ] Recommendations documented
