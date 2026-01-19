# RDS Alert Investigation SOP v2.0

**Version**: 2.0
**Updated**: 2025-01-19
**Scope**: AWS RDS Database Alert Diagnosis (MySQL & PostgreSQL)
**Integration**: MCP with 60+ MySQL, 3 PostgreSQL Servers

---

## Executive Summary

This Standard Operating Procedure provides a systematic 8-phase methodology for investigating AWS RDS database alerts. Version 2.0 introduces parallel query execution, enhanced lock detection, and PostgreSQL support.

---

## Table of Contents

1. [Alert Categories & Priorities](#1-alert-categories--priorities)
2. [Phase 0: Alert Validation](#phase-0-alert-validation)
3. [Phase 1: Data Availability Check](#phase-1-data-availability-check)
4. [Phase 2: Connection Analysis](#phase-2-connection-analysis)
5. [Phase 3: Query Performance](#phase-3-query-performance)
6. [Phase 4: Storage & Replication](#phase-4-storage--replication)
7. [Phase 5: Related Cache Analysis](#phase-5-related-cache-analysis)
8. [Phase 6: Related Service Analysis](#phase-6-related-service-analysis)
9. [Phase 7: CloudWatch Integration](#phase-7-cloudwatch-integration)
10. [Phase 8: Report Generation](#phase-8-report-generation)
11. [Appendix: Data Source Reference](#appendix-data-source-reference)

---

## 1. Alert Categories & Priorities

### Alert Type Matrix

| Category | Metric Pattern | Warning | Critical | Default Priority |
|----------|---------------|---------|----------|------------------|
| **Connections** | threads_connected | > 70% | > 85% | L0/L1 |
| **CPU** | CPUUtilization | > 70% | > 85% | L1 |
| **Slow Queries** | slow_queries/s | > 5/s | > 20/s | L1 |
| **Replication Lag** | Seconds_Behind_Master | > 30s | > 120s | L0 |
| **Storage** | FreeStorageSpace | < 20% | < 10% | L0/L1 |
| **Deadlocks** | innodb_deadlocks | > 0 | > 5/min | L1 |

### Database Tier Classification

| Tier | Databases | Priority | Response SLA |
|------|-----------|----------|--------------|
| **Core** | order_db, payment_db, user_db | L0 | 5-15 min |
| **Business** | inventory_db, promotion_db | L1 | 15-30 min |
| **Supporting** | analytics_db, log_db | L2 | 30min-2hr |

---

## Phase 0: Alert Validation

### Objective
Verify alert accuracy and extract investigation context.

### Metadata Extraction

```
RDS Instance ID: _______________
Endpoint: _____________________
Engine: MySQL / PostgreSQL
Version: ______________________
Database: _____________________
Alert Type: ___________________
Alert Value: __________________
Threshold: ____________________
Duration: _____________________
```

### Database Tier Lookup

```sql
SELECT db.database_name, db.service_tier, db.owner_team,
       s.service_name, s.service_priority
FROM databases db
JOIN service_db_mapping sdm ON db.id = sdm.db_id
JOIN services s ON sdm.service_id = s.id
WHERE db.endpoint LIKE '%$RDS_INSTANCE%';
```

---

## Phase 1: Data Availability Check

### Prometheus Connectivity

**Primary Datasource**: UMBQuerier-Luckin (UID: `df8o21agxtkw0d`)

```promql
# Verify MySQL exporter is reporting
up{job="mysql-exporter", instance=~"$DB_HOST.*"}

# Check data freshness
mysql_up{instance=~"$DB_HOST.*"}
```

### Direct Database Connectivity

Test via mcp-db-gateway:
```sql
SELECT 1 AS connectivity_test;
SELECT @@hostname, @@port, @@version;
```

---

## Phase 2: Connection Analysis

### Objective
Analyze connection pool status and identify issues.

### MySQL Connection Queries (Parallel)

```sql
-- Overall connection status
SELECT
    @@max_connections AS max_conn,
    (SELECT COUNT(*) FROM information_schema.processlist) AS current_conn,
    (SELECT COUNT(*) FROM information_schema.processlist WHERE command='Sleep') AS idle_conn,
    (SELECT COUNT(*) FROM information_schema.processlist WHERE command!='Sleep') AS active_conn;

-- Connections by user and host
SELECT user, host, COUNT(*) AS connections,
       SUM(IF(command='Sleep', 1, 0)) AS idle,
       SUM(IF(command!='Sleep', 1, 0)) AS active
FROM information_schema.processlist
GROUP BY user, host
ORDER BY connections DESC
LIMIT 20;

-- Connection state breakdown
SELECT command, state, COUNT(*) AS count
FROM information_schema.processlist
GROUP BY command, state
ORDER BY count DESC;
```

### PostgreSQL Connection Queries

```sql
-- Overall connection status
SELECT
    (SELECT setting::int FROM pg_settings WHERE name='max_connections') AS max_conn,
    COUNT(*) AS current_conn,
    SUM(CASE WHEN state='idle' THEN 1 ELSE 0 END) AS idle_conn,
    SUM(CASE WHEN state='active' THEN 1 ELSE 0 END) AS active_conn
FROM pg_stat_activity WHERE datname = current_database();

-- Connections by user and client
SELECT usename, client_addr, state, COUNT(*) AS connections
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY usename, client_addr, state
ORDER BY connections DESC;
```

### Connection Health Matrix

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Utilization | < 70% | 70-85% | > 85% |
| Idle Connections | < 50% | 50-70% | > 70% |
| Active Queries | < 50 | 50-100 | > 100 |

---

## Phase 3: Query Performance

### Objective
Identify slow and problematic queries.

### MySQL Slow Query Detection

```sql
-- Long-running queries (> 5 seconds)
SELECT id, user, host, db, time, state,
       LEFT(info, 200) AS query_preview
FROM information_schema.processlist
WHERE command != 'Sleep' AND time > 5
ORDER BY time DESC
LIMIT 20;

-- Recent slow queries from slow log
SELECT start_time, user_host, query_time, lock_time,
       rows_sent, rows_examined, LEFT(sql_text, 200)
FROM mysql.slow_log
WHERE start_time > NOW() - INTERVAL 1 HOUR
ORDER BY query_time DESC
LIMIT 20;
```

### MySQL Lock Detection

```sql
-- Current lock waits
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;

-- Metadata locks
SELECT * FROM performance_schema.metadata_locks
WHERE LOCK_STATUS = 'PENDING';
```

### PostgreSQL Query Analysis

```sql
-- Long-running queries
SELECT pid, usename, query_start, state,
       LEFT(query, 200) AS query_preview,
       EXTRACT(EPOCH FROM now() - query_start) AS duration_sec
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < now() - interval '5 seconds'
ORDER BY query_start
LIMIT 20;

-- Lock waits
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.usename AS blocked_user,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.usename AS blocking_user,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted AND blocking_locks.granted;
```

---

## Phase 4: Storage & Replication

### Storage Analysis

**MySQL Table Sizes**:
```sql
SELECT table_schema, table_name,
       ROUND(data_length/1024/1024, 2) AS data_mb,
       ROUND(index_length/1024/1024, 2) AS index_mb,
       ROUND((data_length + index_length)/1024/1024, 2) AS total_mb,
       table_rows
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
ORDER BY (data_length + index_length) DESC
LIMIT 20;
```

**PostgreSQL Table Sizes**:
```sql
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS total_size,
       n_live_tup, n_dead_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_ratio
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||relname) DESC
LIMIT 20;
```

### Replication Status

**MySQL**:
```sql
SHOW SLAVE STATUS\G
-- Key fields: Seconds_Behind_Master, Slave_IO_Running, Slave_SQL_Running
```

**PostgreSQL**:
```sql
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

---

## Phase 5: Related Cache Analysis

### Objective
Check if cache issues are contributing to database load.

### Find Related Redis Clusters

```sql
SELECT rc.cluster_name, rc.endpoint, srm.cache_purpose
FROM service_redis_mapping srm
JOIN redis_clusters rc ON srm.redis_id = rc.id
JOIN service_db_mapping sdm ON srm.service_id = sdm.service_id
WHERE sdm.db_host = '$DB_HOST';
```

### Redis Health Check

```promql
# Memory usage
redis_memory_used_bytes{cluster="$CLUSTER"} / redis_memory_max_bytes{cluster="$CLUSTER"} * 100

# Hit rate
rate(redis_keyspace_hits_total{cluster="$CLUSTER"}[5m]) /
(rate(redis_keyspace_hits_total{cluster="$CLUSTER"}[5m]) +
 rate(redis_keyspace_misses_total{cluster="$CLUSTER"}[5m]))
```

**Low cache hit rate → more database load**

---

## Phase 6: Related Service Analysis

### Objective
Identify services impacted by database issues.

### Find Dependent Services

```sql
SELECT s.service_name, s.service_priority, s.instance_count,
       ec.instance_ip
FROM services s
JOIN service_db_mapping sdm ON s.id = sdm.service_id
LEFT JOIN ec2_instances ec ON s.id = ec.service_id
WHERE sdm.db_host = '$DB_HOST';
```

### Service Health Check

```promql
# HTTP error rate for dependent services
sum(rate(http_server_requests_seconds_count{service=~"$DEPENDENT_SERVICES",status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count{service=~"$DEPENDENT_SERVICES"}[5m]))

# Response time
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{service=~"$DEPENDENT_SERVICES"}[5m]))
```

---

## Phase 7: CloudWatch Integration

### RDS CloudWatch Metrics

Query these metrics:
- `DatabaseConnections`
- `CPUUtilization`
- `FreeStorageSpace`
- `ReadIOPS`, `WriteIOPS`
- `ReadLatency`, `WriteLatency`
- `ReplicaLag` (for replicas)

### CloudWatch Log Insights

```sql
-- Error log analysis
fields @timestamp, @message
| filter @logStream like '$RDS_INSTANCE'
| filter @message like /error|warning|deadlock/i
| sort @timestamp desc
| limit 50
```

---

## Phase 8: Report Generation

### Standard Report Template

```markdown
# RDS Alert Investigation Report

**Report ID**: [Auto-generated]
**Timestamp**: [Investigation time]

## 1. Alert Summary

| Field | Value |
|-------|-------|
| RDS Instance | [instance-id] |
| Engine | [MySQL/PostgreSQL] [version] |
| Database | [database-name] |
| Tier | [Core/Business/Supporting] |
| Alert Type | [Category] |
| Priority | [L0/L1/L2] |

## 2. Connection Status

| Metric | Current | Max | Utilization |
|--------|---------|-----|-------------|
| Connections | X | Y | Z% |
| Active | N | - | - |
| Idle | M | - | - |

### Top Connection Sources
| User | Host | Connections | Active |
|------|------|-------------|--------|
| [user] | [host] | N | M |

## 3. Query Analysis

### Long-Running Queries
| ID | User | Time | Query Preview |
|----|------|------|---------------|
| [id] | [user] | Xs | [query...] |

### Lock Waits
| Waiting | Blocking | Duration |
|---------|----------|----------|
| [thread] | [thread] | Xs |

## 4. Storage Status

| Database/Table | Size | Rows | Growth |
|----------------|------|------|--------|
| [name] | X GB | N | +Y%/day |

## 5. Replication Status

| Metric | Value | Status |
|--------|-------|--------|
| Lag | Xs | [OK/WARN/CRIT] |
| IO Thread | Running | OK |
| SQL Thread | Running | OK |

## 6. Dependency Status

### Related Cache
| Cluster | Memory | Hit Rate | Status |
|---------|--------|----------|--------|
| [name] | X% | Y% | [OK/WARN] |

### Dependent Services
| Service | Priority | Error Rate | Status |
|---------|----------|------------|--------|
| [name] | [L0/L1] | X% | [OK/WARN] |

## 7. Root Cause Analysis

**Primary Cause**: [Determined cause]

**Evidence**:
1. [Evidence with query/metric reference]
2. [Evidence]

## 8. Recommendations

| Priority | Action | Owner |
|----------|--------|-------|
| **Immediate** | [Action] | [Team] |
| **Short-term** | [Action] | [Team] |
| **Long-term** | [Action] | [Team] |
```

---

## Appendix: Data Source Reference

### MySQL Servers by Domain

| Domain | Count | Key Databases |
|--------|-------|---------------|
| Sales/CRM | 15 | order_db, payment_db |
| SCM | 12 | inventory_db, warehouse_db |
| Operations | 10 | shop_db, delivery_db |
| DevOps | 8 | monitor_db, config_db |
| Finance | 6 | finance_db, billing_db |

### PostgreSQL Servers

| Server | Purpose | Priority |
|--------|---------|----------|
| postgres-main | Dify, analytics | L1 |
| postgres-log | Log aggregation | L2 |
| postgres-report | Reporting | L2 |

### Service Owner Contacts

| Database Tier | Contact | Escalation |
|---------------|---------|------------|
| Core (L0) | dba-team@ | CTO |
| Business (L1) | dba-team@ | VP Eng |
| Supporting (L2) | dba-team@ | Eng Manager |

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.0 | 2025-01-19 | DevOps | PostgreSQL support, lock detection |
| 1.0 | 2025-01-11 | DevOps | Initial release |
