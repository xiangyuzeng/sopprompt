# Redis Alert Investigation SOP v1.0

**Version**: 1.0
**Updated**: 2025-01-19
**Scope**: AWS ElastiCache Redis Alert Diagnosis
**Integration**: MCP with 74 Redis Clusters

---

## Executive Summary

This Standard Operating Procedure provides a systematic 7-phase methodology for investigating Redis/ElastiCache alerts including memory pressure, latency spikes, and connection issues.

---

## Table of Contents

1. [Alert Categories & Priorities](#1-alert-categories--priorities)
2. [Phase 0: Alert Validation](#phase-0-alert-validation)
3. [Phase 1: Data Availability Check](#phase-1-data-availability-check)
4. [Phase 2: Health Assessment](#phase-2-health-assessment)
5. [Phase 3: Direct Diagnostics](#phase-3-direct-diagnostics)
6. [Phase 4: Slow Command Analysis](#phase-4-slow-command-analysis)
7. [Phase 5: Service Dependency](#phase-5-service-dependency)
8. [Phase 6: CloudWatch Integration](#phase-6-cloudwatch-integration)
9. [Phase 7: Report Generation](#phase-7-report-generation)
10. [Appendix: Data Source Reference](#appendix-data-source-reference)

---

## 1. Alert Categories & Priorities

### Alert Type Matrix

| Category | Metric | Warning | Critical | Default Priority |
|----------|--------|---------|----------|------------------|
| **Memory** | used_memory/maxmemory | > 75% | > 85% | L0/L1 |
| **CPU** | EngineCPUUtilization | > 60% | > 80% | L1 |
| **Connections** | connected_clients | > 80% max | > 90% max | L0/L1 |
| **Latency** | instantaneous_ops_per_sec latency | > 2ms avg | > 5ms avg | L1/L2 |
| **Evictions** | evicted_keys | > 100/min | > 500/min | L0/L1 |
| **Replication** | master_link_status | - | down | L0 |

### Cluster Tier Classification

| Tier | Clusters | Priority | Response SLA |
|------|----------|----------|--------------|
| **Session** | session-cache, auth-cache | L0 | 5-15 min |
| **Business** | product-cache, cart-cache | L1 | 15-30 min |
| **Supporting** | config-cache, analytics-cache | L2 | 30min-2hr |

---

## Phase 0: Alert Validation

### Objective
Verify alert accuracy and extract investigation context.

### Metadata Extraction

```
Cluster Name: _________________
Node Endpoint: ________________
Node Role: Primary / Replica
Alert Type: ___________________
Alert Value: __________________
Threshold: ____________________
Duration: _____________________
```

### Cluster Tier Lookup

```sql
SELECT rc.cluster_name, rc.tier, rc.max_memory_gb,
       s.service_name, s.service_priority
FROM redis_clusters rc
JOIN service_redis_mapping srm ON rc.id = srm.redis_id
JOIN services s ON srm.service_id = s.id
WHERE rc.cluster_name = '$CLUSTER_NAME';
```

---

## Phase 1: Data Availability Check

### Prometheus Connectivity

**Primary Datasource**: prometheus_redis (UID: `ff6p0gjt24phce`)

```promql
# Verify Redis exporter is reporting
up{job="redis-exporter", cluster="$CLUSTER"}

# Check data freshness
redis_up{cluster="$CLUSTER"}
```

### Direct Redis Access

Test via mcp-db-gateway:
```
redis_command(server="$CLUSTER", command="PING", args=[])
```

---

## Phase 2: Health Assessment

### Objective
Collect comprehensive cluster health metrics.

### Prometheus Queries (Execute in Parallel)

**Memory Metrics**:
```promql
# Memory usage
redis_memory_used_bytes{cluster="$CLUSTER"}
redis_memory_max_bytes{cluster="$CLUSTER"}

# Fragmentation ratio (> 1.5 is concerning)
redis_memory_fragmentation_ratio{cluster="$CLUSTER"}

# Evictions
rate(redis_evicted_keys_total{cluster="$CLUSTER"}[5m])
```

**CPU & Performance**:
```promql
# Operations per second
redis_instantaneous_ops_per_sec{cluster="$CLUSTER"}

# Command latency
redis_commands_duration_seconds_total{cluster="$CLUSTER"} / redis_commands_total{cluster="$CLUSTER"}
```

**Connections**:
```promql
# Client connections
redis_connected_clients{cluster="$CLUSTER"}

# Blocked clients (waiting on BLPOP etc)
redis_blocked_clients{cluster="$CLUSTER"}

# Rejected connections
rate(redis_rejected_connections_total{cluster="$CLUSTER"}[5m])
```

**Network**:
```promql
# Network I/O
rate(redis_net_input_bytes_total{cluster="$CLUSTER"}[5m])
rate(redis_net_output_bytes_total{cluster="$CLUSTER"}[5m])
```

### Health Status Matrix

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Memory | < 75% | 75-85% | > 85% |
| Fragmentation | < 1.3 | 1.3-1.5 | > 1.5 |
| Latency (avg) | < 1ms | 1-2ms | > 2ms |
| Blocked Clients | 0 | 1-5 | > 5 |
| Evictions/min | 0 | 1-100 | > 100 |

---

## Phase 3: Direct Diagnostics

### Objective
Execute Redis commands for detailed analysis.

### INFO Command

```
redis_command(server="$CLUSTER", command="INFO", args=["all"])
```

**Key Sections to Analyze**:

| Section | Key Metrics | Concern Threshold |
|---------|-------------|-------------------|
| Memory | used_memory, maxmemory, mem_fragmentation_ratio | frag > 1.5 |
| Clients | connected_clients, blocked_clients | blocked > 0 |
| Stats | instantaneous_ops_per_sec, evicted_keys | evictions > 0 |
| Replication | role, master_link_status, master_last_io_seconds_ago | link down |
| CPU | used_cpu_sys, used_cpu_user | high combined |

### MEMORY Commands

```
# Memory breakdown
redis_command(server="$CLUSTER", command="MEMORY", args=["STATS"])

# Memory health check
redis_command(server="$CLUSTER", command="MEMORY", args=["DOCTOR"])
```

### CLIENT Commands

```
# List all client connections
redis_command(server="$CLUSTER", command="CLIENT", args=["LIST"])
```

Analyze for:
- Connection distribution across services
- Long idle connections
- Clients in specific states (blocked, etc.)

---

## Phase 4: Slow Command Analysis

### Objective
Identify problematic commands causing latency.

### SLOWLOG Analysis

```
redis_command(server="$CLUSTER", command="SLOWLOG", args=["GET", "20"])
```

**Analyze for**:
- Commands taking > 10ms
- Frequent O(N) operations: KEYS, SMEMBERS on large sets, HGETALL on large hashes
- Large key operations

### Common Slow Command Patterns

| Command | Issue | Fix |
|---------|-------|-----|
| KEYS * | O(N) scan | Use SCAN |
| SMEMBERS large_set | O(N) | Paginate or restructure |
| HGETALL large_hash | O(N) | Use HSCAN or partial reads |
| SORT | O(N+M*log(M)) | Add LIMIT, use sorted sets |
| DEL large_key | Blocking | Use UNLINK |

### Big Keys Detection

```
# Check key count by type
redis_command(server="$CLUSTER", command="INFO", args=["keyspace"])

# Sample large keys (use cautiously)
redis_command(server="$CLUSTER", command="DEBUG", args=["OBJECT", "ENCODING", "$KEY"])
```

---

## Phase 5: Service Dependency

### Objective
Map affected services and check their health.

### Find Dependent Services

```sql
SELECT s.service_name, s.service_priority, s.instance_count,
       srm.cache_purpose
FROM services s
JOIN service_redis_mapping srm ON s.id = srm.service_id
WHERE srm.cluster_name = '$CLUSTER_NAME';
```

### Check Service Health

```promql
# Cache hit rate for services using this cluster
rate(redis_cache_hits_total{cache_cluster="$CLUSTER"}[5m]) /
(rate(redis_cache_hits_total{cache_cluster="$CLUSTER"}[5m]) +
 rate(redis_cache_misses_total{cache_cluster="$CLUSTER"}[5m]))

# Service error rates
sum(rate(http_server_requests_seconds_count{service=~"$DEPENDENT_SERVICES",status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count{service=~"$DEPENDENT_SERVICES"}[5m]))
```

---

## Phase 6: CloudWatch Integration

### ElastiCache CloudWatch Metrics

Query these metrics:
- `CPUUtilization`
- `EngineCPUUtilization`
- `DatabaseMemoryUsagePercentage`
- `CurrConnections`
- `NetworkBytesIn`, `NetworkBytesOut`
- `ReplicationLag` (for replicas)
- `Evictions`

### Check for AWS Events

- Maintenance windows
- Node replacements
- Cluster modifications

---

## Phase 7: Report Generation

### Standard Report Template

```markdown
# Redis Alert Investigation Report

**Report ID**: [Auto-generated]
**Timestamp**: [Investigation time]

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Cluster | [cluster-name] |
| Node | [Primary/Replica] [endpoint] |
| Tier | [Session/Business/Supporting] |
| Alert Type | [Category] |
| Priority | [L0/L1/L2] |

## 2. Cluster Health

| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Memory Used | X% | 85% | [OK/WARN/CRIT] |
| Fragmentation | X.XX | 1.5 | [OK/WARN/CRIT] |
| Connections | N | M max | [OK/WARN/CRIT] |
| Latency (avg) | Xms | 2ms | [OK/WARN/CRIT] |
| Ops/sec | N | - | - |
| Evictions/min | N | 100 | [OK/WARN/CRIT] |

## 3. Memory Analysis

| Metric | Value |
|--------|-------|
| Used Memory | X MB |
| Max Memory | Y MB |
| Fragmentation Ratio | X.XX |
| Evicted Keys (1h) | N |

## 4. Slow Commands (Last 10min)

| Command | Duration | Key Pattern | Count |
|---------|----------|-------------|-------|
| [CMD] | Xms | [pattern] | N |

## 5. Client Connections

| Source | Connections | Idle | Purpose |
|--------|-------------|------|---------|
| [service] | N | M | [cache type] |

## 6. Affected Services

| Service | Priority | Hit Rate | Error Rate | Status |
|---------|----------|----------|------------|--------|
| [name] | [L0/L1] | X% | Y% | [OK/WARN] |

## 7. Root Cause Analysis

**Primary Cause**: [Determined cause]

**Evidence**:
1. [Evidence with metric/command reference]
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

### Prometheus Datasource

| Name | UID | Purpose |
|------|-----|---------|
| prometheus_redis | `ff6p0gjt24phce` | Redis metrics |

### Redis Clusters by Domain

| Domain | Cluster Count | Examples |
|--------|---------------|----------|
| Session/Auth | 8 | session-cache, auth-cache |
| Product | 12 | product-cache, catalog-cache |
| Order/Cart | 10 | cart-cache, order-cache |
| Config | 5 | config-cache |
| Analytics | 6 | analytics-cache |

### Root Cause Quick Reference

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| High memory + evictions | Key TTL too long | Review TTL strategy |
| High CPU + slow ops | O(N) commands | Optimize commands |
| Fragmentation > 1.5 | Key size variations | MEMORY PURGE or restart |
| Blocked clients | BLPOP timeouts | Check producer/consumer |
| Connection spikes | Pool misconfiguration | Tune pool settings |

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-19 | DevOps | Initial release |
