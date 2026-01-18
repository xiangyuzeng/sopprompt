# RDS Alert Investigation - Quick Prompt v1.0
## Copy → Fill Placeholders → Run in Claude Code

---

```markdown
# RDS Alert Investigation - Database Diagnosis

## Alert Context
- **Alert Name:** [PASTE_ALERT_NAME]
- **Metric:** [PASTE_METRIC]
- **Time:** [YYYY-MM-DD HH:MM TZ] (UTC: [XX:XX])
- **RDS Instance:** [aws-luckyus-xxx-rw]
- **Database:** [luckyus_xxx]
- **Status:** [Active/Resolved]
- **Environment:** Luckin-PROD

---

## Phase 0: Instance Identification

Determine:
- Database type: MySQL / PostgreSQL
- Service level: L0 (sales*, payment*, opshop*) / L1 (auth*, mdm*) / L2 (devops*, ldas*)
- Connected services (from architecture)
- DB owner

---

## Phase 1: Data Check

**MySQL** (aws-luckyus-devops-rw):
```sql
SELECT MIN(create_time), MAX(create_time) 
FROM t_umb_alert_log WHERE instance LIKE '%[RDS_INSTANCE]%';
```

---

## Phase 2: Database Health (RUN ALL)

### 2.1 Connections 🔴 CHECK FIRST
**Direct Query on RDS Instance:**
```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';

-- Connections by host
SELECT user, host, db, COUNT(*) as conn, SUM(IF(command='Sleep',1,0)) as sleeping
FROM information_schema.processlist
GROUP BY user, host, db ORDER BY conn DESC;

-- Long-running connections (>5 min)
SELECT id, user, host, db, command, time, LEFT(info,100)
FROM information_schema.processlist WHERE time > 300 ORDER BY time DESC;
```

**Prometheus** (UMBQuerier-Luckin):
```promql
mysql_global_status_threads_connected{instance=~".*[RDS_INSTANCE].*"} / mysql_global_variables_max_connections * 100
```

### 2.2 Slow Queries
```sql
SELECT start_time, user_host, query_time, rows_examined, LEFT(sql_text,200)
FROM mysql.slow_log 
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY query_time DESC LIMIT 20;
```

**Prometheus:**
```promql
rate(mysql_global_status_slow_queries{instance=~".*[RDS_INSTANCE].*"}[5m])
```

### 2.3 Storage
```sql
-- Database sizes
SELECT table_schema, ROUND(SUM(data_length+index_length)/1024/1024/1024,2) AS 'GB'
FROM information_schema.tables GROUP BY table_schema ORDER BY GB DESC;

-- Binary logs
SHOW BINARY LOGS;
SHOW VARIABLES LIKE 'expire_logs_days';
```

**AWS CloudWatch:** FreeStorageSpace, BinLogDiskUsage

### 2.4 Replication (if replica)
```sql
SHOW REPLICA STATUS\G
-- Check: Seconds_Behind_Master, Slave_IO_Running, Slave_SQL_Running
```

**Prometheus:**
```promql
mysql_slave_status_seconds_behind_master{instance=~".*[RDS_INSTANCE].*"}
```

### 2.5 Locks
```sql
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
SHOW ENGINE INNODB STATUS\G
```

---

## Phase 3: Related Cache (Redis)

| MySQL DB | Redis Cluster |
|----------|---------------|
| luckyus_sales_crm | luckyus-isales-crm |
| luckyus_sales_order | luckyus-isales-order |
| luckyus_scm_shopstock | luckyus-scm-shopstock |
| luckyus_auth | luckyus-auth |

**Redis Check:**
```
INFO memory
INFO clients
SLOWLOG GET 10
```

**Prometheus** (prometheus_redis):
```promql
redis_memory_used_bytes{instance=~".*[REDIS_CLUSTER].*"}
redis_connected_clients{instance=~".*[REDIS_CLUSTER].*"}
```

---

## Phase 4: Alert History

**MySQL** (aws-luckyus-devops-rw):
```sql
SELECT alert_name, alert_status, create_time, resolve_time
FROM t_umb_alert_log
WHERE instance LIKE '%[RDS_INSTANCE]%' OR instance LIKE '%[DB_NAME]%'
ORDER BY create_time DESC LIMIT 20;
```

---

## Output Format

### Summary
| Field | Value |
|-------|-------|
| Root Cause | |
| RDS Instance | |
| Database | |
| Service Level | L0/L1/L2 |
| DB Owner | |
| Resolution | |

### Database Health
| Resource | Status | Current | Threshold |
|----------|--------|---------|-----------|
| Connections | ✅/⚠️/🔴 | / max | 80% |
| Slow Queries | ✅/⚠️/🔴 | /hour | |
| Storage | ✅/⚠️/🔴 | GB free | <10GB |
| Replication | ✅/⚠️/🔴 | sec lag | <60s |

### Related Cache
| Redis | Status | Memory | Connections |
|-------|--------|--------|-------------|
| | ✅/⚠️/🔴 | | |

### Recommendations
```

---

## Quick Reference

### RDS Instance to DB Mapping
| Instance | Database | Service Level | DB Owner |
|----------|----------|---------------|----------|
| aws-luckyus-salescrm-rw | luckyus_sales_crm | L0 | 钟志 |
| aws-luckyus-salesorder-rw | luckyus_sales_order | L0 | 钟志 |
| aws-luckyus-opshop-rw | luckyus_opshop | L0 | 钟志 |
| aws-luckyus-scm-shopstock-rw | luckyus_scm_shopstock | L0 | 钟志 |
| aws-luckyus-ldas-rw | luckyus_ldas* | L2 | 李经宇 |
| aws-luckyus-devops-rw | luckyus_devops | L2 | 李培龙 |
| aws-luckyus-dify-rw | dify (PostgreSQL) | L2 | |

### Prometheus Datasources
| Name | UID | Use |
|------|-----|-----|
| UMBQuerier-Luckin | df8o21agxtkw0d | MySQL metrics |
| prometheus_redis | ff6p0gjt24phce | Redis metrics |

### Common Alert Types
| Alert | Quick Check | Common Cause |
|-------|-------------|--------------|
| 连接数 | Threads_connected | Connection leak |
| 慢查询 | mysql.slow_log | Missing index |
| 磁盘 | CloudWatch Storage | Binary logs |
| 复制延迟 | SHOW REPLICA STATUS | Heavy writes |

---

## Checklist

- [ ] RDS instance identified
- [ ] Service level determined
- [ ] Connected services listed
- [ ] Connections checked
- [ ] Slow queries reviewed
- [ ] Storage verified
- [ ] Replication checked (if applicable)
- [ ] Related Redis cache checked
- [ ] Alert history correlated
- [ ] Root cause identified
- [ ] DB owner notified
