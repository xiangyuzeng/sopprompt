# RDS Alert Investigation - Quick Prompt v2.0

**Purpose**: Rapid RDS database alert diagnosis template for Claude Code with MCP integration.

---

## Alert Context

```
RDS Instance: ________________
Endpoint: ____________________
Engine: MySQL / PostgreSQL
Database: ____________________
Database Tier: Core / Business / Supporting
Service Priority: L0 / L1 / L2
Alert Type: __________________
Alert Value: _________________
```

---

## Phase 0: Quick Classification

| Alert Type | Priority | First Action |
|------------|----------|--------------|
| Connections High | L0/L1 | Check connection by user/host |
| CPU High | L1 | Find expensive queries |
| Slow Queries | L1 | Check slow log |
| Replication Lag | L0 | Check slave status |
| Storage Low | L0/L1 | Find large tables |
| Deadlocks | L1 | Check lock waits |

---

## Phase 1: Connection Analysis

### MySQL
```sql
-- Overall status
SELECT @@max_connections, COUNT(*) AS current
FROM information_schema.processlist;

-- By user/host
SELECT user, host, COUNT(*) AS connections,
       SUM(IF(command='Sleep', 1, 0)) AS idle
FROM information_schema.processlist
GROUP BY user, host ORDER BY connections DESC LIMIT 10;
```

### PostgreSQL
```sql
SELECT usename, client_addr, state, COUNT(*)
FROM pg_stat_activity WHERE datname = current_database()
GROUP BY usename, client_addr, state ORDER BY count DESC;
```

---

## Phase 2: Query Performance

### MySQL Long Queries
```sql
SELECT id, user, time, LEFT(info, 150) AS query
FROM information_schema.processlist
WHERE command != 'Sleep' AND time > 5
ORDER BY time DESC LIMIT 10;
```

### MySQL Lock Detection
```sql
SELECT r.trx_mysql_thread_id AS waiting, b.trx_mysql_thread_id AS blocking
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

### PostgreSQL Long Queries
```sql
SELECT pid, usename, EXTRACT(EPOCH FROM now() - query_start) AS sec,
       LEFT(query, 150) FROM pg_stat_activity
WHERE state = 'active' AND query_start < now() - interval '5 seconds';
```

---

## Phase 3: Storage Check

### MySQL
```sql
SELECT table_schema, table_name,
       ROUND((data_length + index_length)/1024/1024, 2) AS total_mb
FROM information_schema.tables
ORDER BY (data_length + index_length) DESC LIMIT 10;
```

### Replication
```sql
SHOW SLAVE STATUS\G  -- Check Seconds_Behind_Master
```

---

## Phase 4: Prometheus Queries

```promql
# Connection utilization
mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100

# Query rate
rate(mysql_global_status_queries[5m])

# Slow queries
rate(mysql_global_status_slow_queries[5m])
```

---

## Quick Reference

### Datasources
| Name | Server/UID | Purpose |
|------|------------|---------|
| MySQL | mcp-db-gateway | Direct queries |
| Prometheus | `df8o21agxtkw0d` | Metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Topology |

### Thresholds
| Metric | Warning | Critical |
|--------|---------|----------|
| Connections | 70% | 85% |
| CPU | 70% | 85% |
| Replication Lag | 30s | 120s |
| Storage Free | 20% | 10% |

### Database Tiers
| Tier | Examples | Priority |
|------|----------|----------|
| Core | order_db, payment_db | L0 |
| Business | inventory_db | L1 |
| Supporting | analytics_db | L2 |

---

## Output Format

```markdown
## RDS Investigation Summary

**Instance**: [RDS Instance ID]
**Engine**: [MySQL/PostgreSQL] [Version]
**Database**: [Name] (Tier: [Core/Business/Supporting])
**Alert**: [Type] - [Value] vs [Threshold]

### Connection Status
| Metric | Current | Max | Utilization |
|--------|---------|-----|-------------|
| Connections | X | Y | Z% |

### Top Connection Sources
| User | Host | Count |
|------|------|-------|

### Query Issues
[Long queries or locks found]

### Root Cause
[Primary cause with evidence]

### Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
```

---

## Checklist

- [ ] Connection status checked
- [ ] Long-running queries identified
- [ ] Lock waits checked
- [ ] Storage/replication verified
- [ ] Related cache health checked
- [ ] Dependent services reviewed
- [ ] Root cause determined

---

## Cross-Skill Integration

- Cache issues → `/investigate-redis`
- Application issues → `/investigate-apm`
- Host issues → `/investigate-ec2`
- K8s pod issues → `/investigate-k8s`
