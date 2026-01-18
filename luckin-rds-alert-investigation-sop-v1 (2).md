# Luckin Coffee DevOps Alert Investigation SOP
## RDS Database Diagnosis Edition v1.0
### Integrated with MCP Data Sources & System Architecture

---

## Document Info

| Field | Value |
|-------|-------|
| Version | 1.0 |
| Last Updated | 2025-01-11 |
| Scope | AWS RDS Alerts (MySQL, PostgreSQL) |
| Related SOPs | EC2 SOP v4.0, K8s SOP v1.0 |

---

## Available MCP Data Sources for RDS Investigation

### MySQL Databases (60 servers)

| Category | Servers | Service Level |
|----------|---------|---------------|
| **DevOps/Platform** | aws-luckyus-devops-rw, aws-luckyus-ldas-rw, aws-luckyus-ldas01-rw, aws-luckyus-dbatest-rw | L1-L2 |
| **Framework** | aws-luckyus-framework01-rw, aws-luckyus-framework02-rw | L1 |
| **Finance** | aws-luckyus-fichargecontrol-rw, aws-luckyus-fitax-rw, aws-luckyus-ifiaccounting-rw, aws-luckyus-ibillingcentersrv-rw | L0-L1 |
| **Sales/CRM** | aws-luckyus-salescrm-rw, aws-luckyus-salesmarketing-rw, aws-luckyus-salesorder-rw, aws-luckyus-salespayment-rw, aws-luckyus-isalescdp-rw | L0 |
| **SCM/Supply Chain** | aws-luckyus-scm-asset-rw, aws-luckyus-scm-ordering-rw, aws-luckyus-scm-plan-rw, aws-luckyus-scm-purchase-rw, aws-luckyus-scm-shopstock-rw, aws-luckyus-scm-wds-rw, aws-luckyus-scmcommodity-rw, aws-luckyus-scmsrm-rw | L0-L1 |
| **Operations** | aws-luckyus-opshop-rw, aws-luckyus-opshopsale-rw, aws-luckyus-opproduction-rw, aws-luckyus-opqualitycontrol-rw, aws-luckyus-oplog-rw, aws-luckyus-opempefficiency-rw | L0-L1 |
| **BigData/Cyber** | aws-luckyus-icyberdata-rw, aws-luckyus-iluckydorisops-rw | L1 |
| **Auth/Security** | aws-luckyus-iluckyauthapi-rw, aws-luckyus-ipermission-rw, aws-luckyus-iriskcontrolservice-rw | L0-L1 |

### PostgreSQL Databases (3 servers)

| Server | Purpose | Service Level |
|--------|---------|---------------|
| aws-luckyus-dify-rw | Dify AI platform | L2 |
| aws-luckyus-difynew-rw | Dify (new instance) | L2 |
| aws-luckyus-pgilkmap-rw | Map service | L2 |

### Prometheus Datasources

| Datasource | UID | Use For |
|------------|-----|---------|
| **UMBQuerier-Luckin** | df8o21agxtkw0d | RDS metrics, custom DB metrics |
| prometheus | ff7hkeec6c9a8e | General metrics |

### Related Redis Clusters (for cache correlation)

| Category | Clusters |
|----------|----------|
| Sales | luckyus-isales-crm, luckyus-isales-order, luckyus-isales-market |
| SCM | luckyus-scm-shopstock, luckyus-scm-commodity |
| Auth | luckyus-auth, luckyus-session |

### AWS Services (via MCP)

| Service | RDS Investigation Use |
|---------|----------------------|
| CloudWatch Metrics | RDS performance metrics |
| CloudWatch Logs | Slow query logs, error logs |
| CloudWatch Alarms | RDS alarms, storage alerts |
| Cost Explorer | RDS cost analysis |

### Grafana Datasources

| Datasource | UID | Purpose |
|------------|-----|---------|
| MySQL-iriskcontrol | af8p2vx4nhce | Risk control DB |
| MySQL-Ldas | ef5ay9lchfg1sa | LDAS platform |
| MySQL-luckyhealth | af8o704xu3280a | Health monitoring |

---

## Lessons Learned from Historical Incidents

| Incident Pattern | Root Cause | What to Check | Fix in SOP |
|------------------|------------|---------------|------------|
| Connection spike | Application connection leak | Processlist, connection pool config | Phase 2.1 |
| Slow query cascade | Missing index, table lock | Slow log, EXPLAIN, lock status | Phase 2.2 |
| Storage exhaustion | Binary logs, large tables | Storage metrics, binlog retention | Phase 2.3 |
| Replication lag | Heavy write load, network | Replica status, binlog position | Phase 2.4 |
| CPU spike | Inefficient query, full scan | Top queries, query plan | Phase 2.5 |
| AWS ES Disk <10G | Elasticsearch storage | ES cluster status | Phase 4 |

---

## Quick Start Checklist

Before investigation:
- [ ] Identify RDS instance name and type (MySQL/PostgreSQL)
- [ ] Determine database service level (L0/L1/L2)
- [ ] Identify connected applications (from architecture)
- [ ] Identify DB owner for escalation
- [ ] Check for related Redis cache
- [ ] List replica instances if any

---

## Service Level Priority Matrix

| Level | Definition | Response Time | Example Databases |
|-------|------------|---------------|-------------------|
| **L0** 🔴 | Core Business | <15 min | luckyus_sales_order, luckyus_sales_payment, luckyus_opshop, luckyus_scm_shopstock |
| **L1** 🟡 | Important | <30 min | luckyus_mdm, luckyus_auth, luckyus_iot_platform, luckyus_upush* |
| **L2** 🟢 | Normal | <2 hours | luckyus_devops, luckyus_izeus, luckyus_ldas* |

---

## Copy-Paste Investigation Prompt

```markdown
# RDS Alert Investigation - Database Diagnosis

## Alert Context
- **Alert Name:** [PASTE_FULL_ALERT_NAME - e.g., 【RDS】P1 连接数大于80%]
- **Alert Metric:** [PASTE_METRIC]
- **Trigger Time:** [YYYY-MM-DD HH:MM TIMEZONE] (UTC: [XX:XX])
- **Duration:** [Duration or "ongoing"]
- **Status:** [Active / Resolved / Flapping]
- **Environment:** Luckin-PROD
- **RDS Instance:** [e.g., aws-luckyus-salescrm-rw]
- **Database Name:** [e.g., luckyus_sales_crm]

---

## PHASE 0: Alert Validation & Instance Identification

### 0.1 Identify RDS Instance Details

**From Alert:**
- RDS Instance Name: [aws-luckyus-xxx-rw]
- Database Type: [MySQL / PostgreSQL]
- Database Name: [luckyus_xxx]
- Is this Read/Write (rw) or Read Replica (ro)?

**Determine Service Level:**

| Database Pattern | Service Level | Response |
|------------------|---------------|----------|
| sales*, payment* | L0 🔴 | <15 min |
| opshop*, opproduction* | L0 🔴 | <15 min |
| scm_shopstock*, scm_commodity* | L0 🔴 | <15 min |
| auth*, mdm*, upush* | L1 🟡 | <30 min |
| devops*, ldas*, izeus* | L2 🟢 | <2 hours |

### 0.2 Identify Connected Applications

**From Architecture - Which services connect to this DB?**

| Database | Connected Services | Service Owner |
|----------|-------------------|---------------|
| luckyus_sales_crm | isalescrmservice, isalescrmadmin | 张晓松 |
| luckyus_sales_order | isalesorderservice, isalesorderadmin | 张晓松 |
| luckyus_opshop | iopshopservice, iopocp | 陈培浩 |
| luckyus_scm_shopstock | iscmshopstock, iscmsims | 方思扬 |
| luckyus_mdm | luckymdm, luckymdmadmin | 林智贤 |

### 0.3 Check for Replica Instances

If alerting instance is primary (rw), check replicas.
If alerting instance is replica (ro), check primary.

```promql
# Find all instances for this database cluster
mysql_up{instance=~".*[DB_NAME].*"}
```

---

## PHASE 1: Data Availability Check

### 1.1 Prometheus Data

**Prometheus** (UMBQuerier-Luckin - UID: df8o21agxtkw0d):
```promql
# Check if RDS metrics exist for the instance
mysql_up{instance=~".*[RDS_INSTANCE].*"}

# Verify data exists for incident time
mysql_global_status_threads_connected{instance=~".*[RDS_INSTANCE].*"}[1d]
```

### 1.2 Alert History

**MySQL** (aws-luckyus-devops-rw):
```sql
-- Check alert history for this RDS instance
SELECT 
    alert_name,
    instance,
    alert_status,
    create_time,
    resolve_time,
    TIMESTAMPDIFF(MINUTE, create_time, COALESCE(resolve_time, NOW())) as duration_min
FROM t_umb_alert_log
WHERE instance LIKE '%[RDS_INSTANCE]%'
   OR alert_name LIKE '%[DB_NAME]%'
ORDER BY create_time DESC
LIMIT 30;
```

### 1.3 AWS CloudWatch

**Via AWS MCP:**
- Check CloudWatch metrics retention
- Get RDS enhanced monitoring data
- Query Performance Insights if enabled

---

## PHASE 2: Cross-System Database Health Check

**⚠️ Run ALL checks regardless of alert type!**

### 2.1 Connection Analysis 🔴 CHECK FIRST FOR CONNECTION ALERTS

**Direct MySQL Query** (Connect to alerting RDS instance):
```sql
-- Current connection status
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW STATUS LIKE 'Threads_running';
SHOW STATUS LIKE 'Connections';

-- Connection limit
SHOW VARIABLES LIKE 'max_connections';

-- Active connections by user/host
SELECT 
    user, 
    host,
    db,
    COUNT(*) as connection_count,
    SUM(IF(command='Sleep', 1, 0)) as sleeping,
    SUM(IF(command!='Sleep', 1, 0)) as active
FROM information_schema.processlist
GROUP BY user, host, db
ORDER BY connection_count DESC;

-- Long-running connections (potential leaks)
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state,
    LEFT(info, 100) as query_preview
FROM information_schema.processlist
WHERE time > 300  -- > 5 minutes
ORDER BY time DESC;

-- Connection by state
SELECT 
    command,
    COUNT(*) as count
FROM information_schema.processlist
GROUP BY command
ORDER BY count DESC;
```

**Prometheus** (UMBQuerier-Luckin):
```promql
# Current connections
mysql_global_status_threads_connected{instance=~".*[RDS_INSTANCE].*"}

# Connection usage percentage
mysql_global_status_threads_connected{instance=~".*[RDS_INSTANCE].*"} 
/ mysql_global_variables_max_connections{instance=~".*[RDS_INSTANCE].*"} * 100

# Connection trend
mysql_global_status_threads_connected{instance=~".*[RDS_INSTANCE].*"}[1h]

# Running threads (active queries)
mysql_global_status_threads_running{instance=~".*[RDS_INSTANCE].*"}

# Aborted connections (connection issues)
rate(mysql_global_status_aborted_connects{instance=~".*[RDS_INSTANCE].*"}[5m])
rate(mysql_global_status_aborted_clients{instance=~".*[RDS_INSTANCE].*"}[5m])

# All RDS instances connection usage
sort_desc(
  mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100
)
```

**AWS CloudWatch Metrics**:
```
Namespace: AWS/RDS
Metrics:
  - DatabaseConnections
  - DBLoadCPU
  - DBLoad
Dimensions:
  - DBInstanceIdentifier: [RDS_INSTANCE]
```

### 2.2 Slow Query Analysis

**Direct MySQL Query**:
```sql
-- Check if slow query log is enabled
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- Recent slow queries from mysql.slow_log (if available)
SELECT 
    start_time,
    user_host,
    query_time,
    lock_time,
    rows_sent,
    rows_examined,
    db,
    LEFT(sql_text, 200) as query_preview
FROM mysql.slow_log
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY query_time DESC
LIMIT 20;

-- Queries currently running for >10 seconds
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state,
    LEFT(info, 200) as query
FROM information_schema.processlist
WHERE command != 'Sleep' 
  AND time > 10
ORDER BY time DESC;
```

**MySQL** (aws-luckyus-ldas-rw or aws-luckyus-ldas01-rw - slow query repository):
```sql
-- If you have a centralized slow query repository
SELECT 
    instance,
    db,
    query_time,
    lock_time,
    rows_examined,
    LEFT(sql_text, 200) as query,
    create_time
FROM slow_query_log_table  -- Adjust table name
WHERE instance LIKE '%[RDS_INSTANCE]%'
  AND create_time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY query_time DESC
LIMIT 20;
```

**Prometheus**:
```promql
# Slow queries rate
rate(mysql_global_status_slow_queries{instance=~".*[RDS_INSTANCE].*"}[5m])

# Slow query trend
increase(mysql_global_status_slow_queries{instance=~".*[RDS_INSTANCE].*"}[1h])
```

### 2.3 Storage Analysis

**Direct MySQL Query**:
```sql
-- Database sizes
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS 'Size (GB)',
    ROUND(SUM(data_free) / 1024 / 1024 / 1024, 2) AS 'Free (GB)'
FROM information_schema.tables
GROUP BY table_schema
ORDER BY SUM(data_length + index_length) DESC;

-- Top 20 largest tables
SELECT 
    table_schema,
    table_name,
    ROUND(data_length / 1024 / 1024 / 1024, 2) AS 'Data (GB)',
    ROUND(index_length / 1024 / 1024 / 1024, 2) AS 'Index (GB)',
    ROUND((data_length + index_length) / 1024 / 1024 / 1024, 2) AS 'Total (GB)',
    table_rows
FROM information_schema.tables
ORDER BY (data_length + index_length) DESC
LIMIT 20;

-- Binary log status (major storage consumer)
SHOW BINARY LOGS;
SHOW VARIABLES LIKE 'expire_logs_days';
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';

-- Temporary table usage
SHOW STATUS LIKE 'Created_tmp%';
```

**AWS CloudWatch Metrics**:
```
Namespace: AWS/RDS
Metrics:
  - FreeStorageSpace
  - BinLogDiskUsage
  - TransactionLogsDiskUsage (PostgreSQL)
  - FreeableMemory
```

**Prometheus**:
```promql
# Binary log space
mysql_global_status_binlog_cache_disk_use{instance=~".*[RDS_INSTANCE].*"}

# InnoDB buffer pool usage
mysql_global_status_innodb_buffer_pool_bytes_data{instance=~".*[RDS_INSTANCE].*"}
```

### 2.4 Replication Analysis (If Applicable)

**Direct MySQL Query on Replica**:
```sql
-- Replica status
SHOW REPLICA STATUS\G

-- Key fields to check:
-- Seconds_Behind_Master (should be 0 or low)
-- Slave_IO_Running (should be Yes)
-- Slave_SQL_Running (should be Yes)
-- Last_Error (should be empty)
```

**Prometheus**:
```promql
# Replication lag
mysql_slave_status_seconds_behind_master{instance=~".*[RDS_INSTANCE].*"}

# Replication status
mysql_slave_status_slave_io_running{instance=~".*[RDS_INSTANCE].*"}
mysql_slave_status_slave_sql_running{instance=~".*[RDS_INSTANCE].*"}

# All replicas with lag
mysql_slave_status_seconds_behind_master > 10
```

**AWS CloudWatch Metrics**:
```
Namespace: AWS/RDS
Metrics:
  - ReplicaLag
  - ReplicationSlotDiskUsage (PostgreSQL)
```

### 2.5 CPU & Performance Analysis

**Direct MySQL Query**:
```sql
-- InnoDB status
SHOW ENGINE INNODB STATUS\G

-- Table locks
SHOW STATUS LIKE 'Table_locks%';

-- InnoDB row locks
SHOW STATUS LIKE 'Innodb_row_lock%';

-- Current locks
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- For MySQL 8.0+
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- Buffer pool status
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

**Prometheus**:
```promql
# CPU usage (if exposed)
mysql_global_status_cpu{instance=~".*[RDS_INSTANCE].*"}

# Questions per second (query load)
rate(mysql_global_status_questions{instance=~".*[RDS_INSTANCE].*"}[5m])

# InnoDB row operations
rate(mysql_global_status_innodb_row_ops_total{instance=~".*[RDS_INSTANCE].*"}[5m])

# Buffer pool hit ratio
mysql_global_status_innodb_buffer_pool_read_requests{instance=~".*[RDS_INSTANCE].*"} 
/ 
(mysql_global_status_innodb_buffer_pool_read_requests{instance=~".*[RDS_INSTANCE].*"} 
 + mysql_global_status_innodb_buffer_pool_reads{instance=~".*[RDS_INSTANCE].*"}) * 100
```

**AWS CloudWatch Metrics**:
```
Namespace: AWS/RDS
Metrics:
  - CPUUtilization
  - ReadIOPS, WriteIOPS
  - ReadLatency, WriteLatency
  - ReadThroughput, WriteThroughput
  - DiskQueueDepth
  - BurstBalance (for gp2/gp3)
```

### 2.6 Query Analysis (For Specific Issues)

**Direct MySQL Query**:
```sql
-- Find queries without proper indexes (full table scans)
SELECT 
    id,
    user,
    host,
    db,
    time,
    LEFT(info, 200) as query
FROM information_schema.processlist
WHERE info IS NOT NULL
  AND info NOT LIKE '%information_schema%'
  AND (info LIKE '%SELECT%' OR info LIKE '%UPDATE%' OR info LIKE '%DELETE%');

-- For identified slow query, run EXPLAIN
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...; -- MySQL 8.0+

-- Index usage statistics (MySQL 8.0+ or via sys schema)
SELECT 
    object_schema,
    object_name,
    index_name,
    count_star as usage_count
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = '[DATABASE]'
ORDER BY count_star DESC;
```

---

## PHASE 3: Related Cache Check (Redis)

**If the database has a related Redis cache, check it too!**

### 3.1 Map Database to Redis Cluster

| MySQL Database | Related Redis Cluster |
|----------------|----------------------|
| luckyus_sales_crm | luckyus-isales-crm |
| luckyus_sales_order | luckyus-isales-order |
| luckyus_sales_marketing | luckyus-isales-market |
| luckyus_scm_shopstock | luckyus-scm-shopstock |
| luckyus_scm_commodity | luckyus-scm-commodity |
| luckyus_auth | luckyus-auth |
| luckyus_session | luckyus-session |
| luckyus_devops | luckyus-devops |
| luckyus_ldas* | luckyus-ldas |

### 3.2 Redis Health Check

**Redis Commands** (on related cluster):
```
# Memory status
INFO memory

# Client connections
INFO clients

# Slow commands
SLOWLOG GET 20

# Hit/miss ratio
INFO stats
# Check keyspace_hits vs keyspace_misses

# Memory fragmentation
INFO memory
# Check mem_fragmentation_ratio
```

**Prometheus** (prometheus_redis - UID: ff6p0gjt24phce):
```promql
# Redis memory
redis_memory_used_bytes{instance=~".*[REDIS_CLUSTER].*"}

# Redis connections
redis_connected_clients{instance=~".*[REDIS_CLUSTER].*"}

# Cache hit ratio
rate(redis_keyspace_hits_total{instance=~".*[REDIS_CLUSTER].*"}[5m]) 
/ 
(rate(redis_keyspace_hits_total{instance=~".*[REDIS_CLUSTER].*"}[5m]) 
 + rate(redis_keyspace_misses_total{instance=~".*[REDIS_CLUSTER].*"}[5m])) * 100
```

**Why check Redis?**
- Cache miss spike → DB connection/query spike
- Cache eviction → Increased DB load
- Cache failure → All traffic hits DB

---

## PHASE 4: Related Service Check

### 4.1 Connected EC2 Instances

For the services that connect to this database, check their health:

**Prometheus** (UMBQuerier-Luckin):
```promql
# Find instances for connected services
# Example: For luckyus_sales_crm, check isalescrmservice
node_uname_info{nodename=~"isalescrmservice.*"}
node_uname_info{nodename=~"isalescrmadmin.*"}

# Check their memory (connection pools live here)
node_memory_app_usage_rate_service_avg{nodename=~"isalescrmservice.*"}

# Check their CPU (query generation)
100 - (avg by(instance) (rate(node_cpu_seconds_total{nodename=~"isalescrmservice.*",mode="idle"}[5m])) * 100)
```

### 4.2 Application Connection Pool Issues

Common patterns:
- **Connection leak**: Connections keep growing without release
- **Pool exhaustion**: All connections in use, app waiting
- **Wrong pool size**: Pool too small for load

**Signs in MySQL**:
```sql
-- Many connections in Sleep state from same host
SELECT host, COUNT(*), AVG(time) as avg_time
FROM information_schema.processlist
WHERE command = 'Sleep'
GROUP BY host
ORDER BY COUNT(*) DESC;

-- If avg_time is high and count is high → potential leak
```

---

## PHASE 5: AWS CloudWatch Deep Dive

### 5.1 RDS Performance Insights (If Enabled)

**Via AWS MCP**:
- Top SQL by load
- Wait events analysis
- Database load breakdown

### 5.2 RDS Event History

**Check for**:
- Failover events
- Parameter changes
- Maintenance events
- Storage autoscaling events

### 5.3 Key CloudWatch Alarms

```
# RDS Alarms to check:
- DatabaseConnections threshold
- CPUUtilization threshold
- FreeStorageSpace threshold
- ReadLatency/WriteLatency threshold
- ReplicaLag threshold
```

---

## PHASE 6: Alert Correlation & History

### 6.1 Alert History

**MySQL** (aws-luckyus-devops-rw):
```sql
-- All alerts for this RDS instance
SELECT 
    alert_name,
    alert_status,
    create_time,
    resolve_time,
    TIMESTAMPDIFF(MINUTE, create_time, COALESCE(resolve_time, NOW())) as duration_min
FROM t_umb_alert_log
WHERE instance LIKE '%[RDS_INSTANCE]%'
   OR instance LIKE '%[DB_NAME]%'
ORDER BY create_time DESC
LIMIT 30;

-- Alerts in same timeframe (correlated issues)
SELECT 
    alert_name,
    instance,
    create_time
FROM t_umb_alert_log
WHERE create_time BETWEEN DATE_SUB('[INCIDENT_TIME]', INTERVAL 30 MINUTE) 
                      AND DATE_ADD('[INCIDENT_TIME]', INTERVAL 30 MINUTE)
ORDER BY create_time;

-- Recurring pattern
SELECT 
    DATE(create_time) as date,
    alert_name,
    COUNT(*) as occurrences
FROM t_umb_alert_log
WHERE instance LIKE '%[DB_NAME]%'
  AND create_time > DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY DATE(create_time), alert_name
ORDER BY date DESC;
```

### 6.2 Correlated EC2/Application Alerts

```sql
-- Check if connected applications also had alerts
SELECT 
    alert_name,
    instance,
    create_time
FROM t_umb_alert_log
WHERE create_time BETWEEN DATE_SUB('[INCIDENT_TIME]', INTERVAL 30 MINUTE) 
                      AND DATE_ADD('[INCIDENT_TIME]', INTERVAL 30 MINUTE)
  AND (instance LIKE '%[SERVICE_NAME]%')
ORDER BY create_time;
```

---

## PHASE 7: PostgreSQL Specific Checks

**For Dify databases (aws-luckyus-dify-rw, aws-luckyus-difynew-rw):**

### 7.1 Connection Status

```sql
-- Current connections
SELECT 
    datname,
    usename,
    application_name,
    client_addr,
    state,
    COUNT(*)
FROM pg_stat_activity
GROUP BY datname, usename, application_name, client_addr, state
ORDER BY COUNT(*) DESC;

-- Connection limit
SHOW max_connections;

-- Active queries
SELECT 
    pid,
    usename,
    datname,
    state,
    query_start,
    NOW() - query_start AS duration,
    LEFT(query, 100) as query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

### 7.2 Lock Analysis

```sql
-- Current locks
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### 7.3 Table Statistics

```sql
-- Table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- Dead tuples (needs VACUUM)
SELECT 
    schemaname,
    relname,
    n_dead_tup,
    n_live_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

---

## PHASE 8: Output Report Format

### Executive Summary
| Field | Value |
|-------|-------|
| **Root Cause** | [One-line summary] |
| **RDS Instance** | [aws-luckyus-xxx-rw] |
| **Database** | [luckyus_xxx] |
| **Service Level** | [L0/L1/L2] |
| **DB Owner** | [Name] |
| **Connected Services** | [List] |
| **Impact** | [What was affected] |
| **Resolution** | [How resolved] |
| **Status** | ✅ Resolved / ⚠️ Monitoring / 🔴 Active |

### Alert Validation
| Item | Alert Said | Actually Found |
|------|------------|----------------|
| Issue Type | [e.g., Connections] | [e.g., Connection leak from app X] |
| Severity | [Alert level] | [Actual impact] |

### Data Sources Used
| Source | Type | Purpose |
|--------|------|---------|
| [RDS Instance] | MySQL | Direct queries |
| UMBQuerier-Luckin | Prometheus | DB metrics |
| aws-luckyus-devops-rw | MySQL | Alert logs |
| [Redis cluster] | Redis | Cache status |
| CloudWatch | AWS | RDS metrics |

### Database Health Check Results
| Resource | Status | Current | Threshold | Findings |
|----------|--------|---------|-----------|----------|
| Connections | ✅/⚠️/🔴 | [X] / [Max] | 80% | |
| Slow Queries | ✅/⚠️/🔴 | [X]/hour | - | |
| Storage | ✅/⚠️/🔴 | [X] GB free | <10GB | |
| Replication Lag | ✅/⚠️/🔴 | [X] seconds | <60s | |
| CPU | ✅/⚠️/🔴 | [X]% | 80% | |
| Buffer Pool | ✅/⚠️/🔴 | [X]% hit | >95% | |

### Related Cache Status
| Redis Cluster | Status | Memory | Connections | Hit Ratio |
|---------------|--------|--------|-------------|-----------|
| [cluster-name] | ✅/⚠️/🔴 | | | |

### Connected Application Status
| Service | Status | Connections to DB | Issues |
|---------|--------|-------------------|--------|
| [service-name] | ✅/⚠️/🔴 | | |

### Top Issues Found
1. [Issue 1 - e.g., Long-running query blocking others]
2. [Issue 2 - e.g., Connection leak from host X]

### Root Cause Analysis
[Detailed explanation]

### Resolution Steps
1. [Step taken]
2. [Step taken]

### Preventive Recommendations

**Priority 1 - Immediate**
| Action | Owner | Impact |
|--------|-------|--------|
| | | |

**Priority 2 - Short Term**
| Action | Owner | Impact |
|--------|-------|--------|

### Monitoring Improvements
- [ ] [New alert rule]
- [ ] [Query optimization]
- [ ] [Index addition]

---

## Appendix A: RDS Instance to Service Mapping

| RDS Instance | Database | Connected Services | Service Owner | DB Owner |
|--------------|----------|-------------------|---------------|----------|
| aws-luckyus-salescrm-rw | luckyus_sales_crm | isalescrmservice, isalescrmadmin | 张晓松 | 钟志 |
| aws-luckyus-salesorder-rw | luckyus_sales_order | isalesorderservice | 张晓松 | 钟志 |
| aws-luckyus-salespayment-rw | luckyus_sales_payment | isalespaymentservice | 张晓松 | 钟志 |
| aws-luckyus-opshop-rw | luckyus_opshop | iopshopservice, iopocp | 陈培浩 | 钟志 |
| aws-luckyus-opproduction-rw | luckyus_opproduction | iopproduction | 陈培浩 | 钟志 |
| aws-luckyus-scm-shopstock-rw | luckyus_scm_shopstock | iscmshopstock, iscmsims | 方思扬 | 钟志 |
| aws-luckyus-scmcommodity-rw | luckyus_scm_commodity | iscmcommodity, iscmcommodityadmin | 方思扬 | 钟志 |
| aws-luckyus-ldas-rw | luckyus_ldas* | ldasnacos, ldascmdb | 孔增 | 李经宇 |
| aws-luckyus-devops-rw | luckyus_devops | DevOps tools | 吴伟根 | 李培龙 |
| aws-luckyus-fitax-rw | luckyus_fi_tax | ifitax | 尤志杰 | 钟志 |

---

## Appendix B: Common RDS Alert Types & Quick Checks

| Alert Type | Quick Check | Common Cause |
|------------|-------------|--------------|
| 连接数 (Connections) | `SHOW STATUS LIKE 'Threads_connected'` | Connection leak, traffic spike |
| 慢查询 (Slow Query) | `SELECT * FROM mysql.slow_log` | Missing index, lock contention |
| 磁盘空间 (Disk Space) | CloudWatch FreeStorageSpace | Binary logs, large tables |
| CPU | CloudWatch CPUUtilization | Inefficient query, full scan |
| 复制延迟 (Replication Lag) | `SHOW REPLICA STATUS` | Heavy writes, network |
| 死锁 (Deadlock) | `SHOW ENGINE INNODB STATUS` | Transaction conflict |

---

## Appendix C: Emergency Resolution Commands

```sql
-- Kill long-running query
KILL [process_id];

-- Kill all connections from specific host (use with caution)
-- First identify:
SELECT id FROM information_schema.processlist WHERE host LIKE '%problem_host%';
-- Then kill each:
KILL [id];

-- Flush hosts (if too many connection errors)
FLUSH HOSTS;

-- Reset query cache (if enabled)
RESET QUERY CACHE;

-- For replication issues, skip error (use with extreme caution)
STOP REPLICA;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START REPLICA;

-- Purge old binary logs (free disk space)
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);
```

---

## Appendix D: Key Prometheus Metrics for RDS

```yaml
# Connections
mysql_global_status_threads_connected
mysql_global_status_threads_running
mysql_global_variables_max_connections
mysql_global_status_aborted_connects
mysql_global_status_aborted_clients

# Query Performance
mysql_global_status_slow_queries
mysql_global_status_questions
mysql_global_status_queries

# InnoDB
mysql_global_status_innodb_buffer_pool_bytes_data
mysql_global_status_innodb_buffer_pool_read_requests
mysql_global_status_innodb_buffer_pool_reads
mysql_global_status_innodb_row_ops_total

# Replication
mysql_slave_status_seconds_behind_master
mysql_slave_status_slave_io_running
mysql_slave_status_slave_sql_running

# Redis (for cache correlation)
redis_memory_used_bytes
redis_connected_clients
redis_keyspace_hits_total
redis_keyspace_misses_total
```

---

## Appendix E: Investigation Completion Checklist

- [ ] **Phase 0:** RDS instance identified, service level determined
- [ ] **Phase 1:** Data availability confirmed
- [ ] **Phase 2:** All database checks completed:
  - [ ] Connections
  - [ ] Slow queries
  - [ ] Storage
  - [ ] Replication (if applicable)
  - [ ] CPU/Performance
  - [ ] Locks
- [ ] **Phase 3:** Related Redis cache checked
- [ ] **Phase 4:** Connected applications checked
- [ ] **Phase 5:** AWS CloudWatch reviewed
- [ ] **Phase 6:** Alert history correlated
- [ ] **Phase 8:** Report generated with:
  - [ ] Root cause identified
  - [ ] Data sources documented
  - [ ] Owner notified
  - [ ] Recommendations provided

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-11 | Initial RDS SOP with MCP integration, MySQL/PostgreSQL coverage |
```
