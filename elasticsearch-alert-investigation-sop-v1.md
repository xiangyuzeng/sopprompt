# Elasticsearch/OpenSearch Alert Investigation SOP v1.0

**Version**: 1.0
**Updated**: 2025-01-19
**Scope**: AWS OpenSearch / Elasticsearch Alert Diagnosis
**Integration**: MCP with Prometheus, CloudWatch

---

## Executive Summary

This Standard Operating Procedure provides a systematic 7-phase methodology for investigating Elasticsearch/OpenSearch alerts including cluster health, index performance, and resource utilization issues.

---

## Table of Contents

1. [Alert Categories & Priorities](#1-alert-categories--priorities)
2. [Phase 0: Alert Validation](#phase-0-alert-validation)
3. [Phase 1: Data Availability Check](#phase-1-data-availability-check)
4. [Phase 2: Cluster Health](#phase-2-cluster-health)
5. [Phase 3: Node Analysis](#phase-3-node-analysis)
6. [Phase 4: Index Performance](#phase-4-index-performance)
7. [Phase 5: Query Analysis](#phase-5-query-analysis)
8. [Phase 6: Service Impact](#phase-6-service-impact)
9. [Phase 7: Report Generation](#phase-7-report-generation)
10. [Appendix: Data Source Reference](#appendix-data-source-reference)

---

## 1. Alert Categories & Priorities

### Alert Type Matrix

| Category | Metric | Warning | Critical | Default Priority |
|----------|--------|---------|----------|------------------|
| **Cluster Health** | cluster_status | yellow | red | L0 |
| **Node Count** | nodes_available | < expected | < quorum | L0 |
| **JVM Heap** | jvm_heap_used | > 75% | > 85% | L0/L1 |
| **CPU** | cpu_utilization | > 70% | > 85% | L1 |
| **Disk** | disk_used | > 75% | > 85% | L0/L1 |
| **Index Performance** | indexing_latency | > 50ms | > 200ms | L1/L2 |
| **Search Performance** | search_latency | > 100ms | > 500ms | L1/L2 |
| **Shard State** | unassigned_shards | > 0 | > 10 | L0/L1 |

### Cluster Tier Classification

| Tier | Clusters | Priority | Response SLA |
|------|----------|----------|--------------|
| **Search** | product-search, order-search | L0 | 5-15 min |
| **Logging** | app-logs, access-logs | L1 | 15-30 min |
| **Analytics** | analytics-es, metrics-es | L2 | 30min-2hr |

---

## Phase 0: Alert Validation

### Objective
Verify alert accuracy and extract investigation context.

### Metadata Extraction

```
Domain/Cluster: _______________
Endpoint: ____________________
Node Role: Master / Data / Ingest / Coordinating
Index Pattern: ________________
Alert Type: ___________________
Alert Value: __________________
Threshold: ____________________
Duration: _____________________
```

### Cluster Tier Lookup

```sql
SELECT ec.cluster_name, ec.tier, ec.node_count,
       s.service_name, s.service_priority
FROM elasticsearch_clusters ec
JOIN service_es_mapping sem ON ec.id = sem.es_cluster_id
JOIN services s ON sem.service_id = s.id
WHERE ec.cluster_name = '$CLUSTER_NAME';
```

---

## Phase 1: Data Availability Check

### Prometheus Connectivity

**Primary Datasource**: UMBQuerier-Luckin (UID: `df8o21agxtkw0d`)

```promql
# Verify ES exporter is reporting
up{job="elasticsearch-exporter", cluster="$CLUSTER"}

# Check data freshness
elasticsearch_cluster_health_status{cluster="$CLUSTER"}
```

### Elasticsearch API Access

If direct API access is available:
```
GET /_cluster/health
GET /_cat/nodes?v
```

---

## Phase 2: Cluster Health

### Objective
Assess overall cluster state and identify issues.

### Prometheus Queries (Execute in Parallel)

**Cluster Status**:
```promql
# Cluster health status (1=green, 2=yellow, 3=red)
elasticsearch_cluster_health_status{cluster="$CLUSTER"}

# Node count
elasticsearch_cluster_health_number_of_nodes{cluster="$CLUSTER"}
elasticsearch_cluster_health_number_of_data_nodes{cluster="$CLUSTER"}

# Shard status
elasticsearch_cluster_health_active_shards{cluster="$CLUSTER"}
elasticsearch_cluster_health_relocating_shards{cluster="$CLUSTER"}
elasticsearch_cluster_health_initializing_shards{cluster="$CLUSTER"}
elasticsearch_cluster_health_unassigned_shards{cluster="$CLUSTER"}
```

**Pending Tasks**:
```promql
elasticsearch_cluster_health_number_of_pending_tasks{cluster="$CLUSTER"}
```

### Cluster Health Matrix

| Status | Meaning | Action |
|--------|---------|--------|
| Green | All shards allocated | Normal |
| Yellow | Primary OK, replicas missing | Check node capacity |
| Red | Primary shards missing | URGENT: Check data nodes |

---

## Phase 3: Node Analysis

### Objective
Analyze individual node health and resource usage.

### Prometheus Queries (Parallel)

**JVM Metrics**:
```promql
# JVM heap usage
elasticsearch_jvm_memory_used_bytes{cluster="$CLUSTER",area="heap"}
  / elasticsearch_jvm_memory_max_bytes{cluster="$CLUSTER",area="heap"} * 100

# GC time
rate(elasticsearch_jvm_gc_collection_seconds_sum{cluster="$CLUSTER"}[5m])

# GC count
rate(elasticsearch_jvm_gc_collection_seconds_count{cluster="$CLUSTER"}[5m])
```

**System Resources**:
```promql
# CPU utilization
elasticsearch_os_cpu_percent{cluster="$CLUSTER"}

# Memory
elasticsearch_os_mem_used_percent{cluster="$CLUSTER"}

# Disk usage
elasticsearch_filesystem_data_size_bytes{cluster="$CLUSTER"}
  - elasticsearch_filesystem_data_free_bytes{cluster="$CLUSTER"}
```

**Thread Pools**:
```promql
# Thread pool rejections
rate(elasticsearch_thread_pool_rejected_count{cluster="$CLUSTER"}[5m])

# Thread pool queue size
elasticsearch_thread_pool_queue_count{cluster="$CLUSTER"}
```

### Node Health Matrix

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| JVM Heap | < 75% | 75-85% | > 85% |
| CPU | < 70% | 70-85% | > 85% |
| Disk | < 75% | 75-85% | > 85% |
| GC Time (5m) | < 5s | 5-10s | > 10s |

---

## Phase 4: Index Performance

### Objective
Analyze index-level metrics and performance.

### Prometheus Queries (Parallel)

**Indexing Performance**:
```promql
# Indexing rate
rate(elasticsearch_indices_indexing_index_total{cluster="$CLUSTER"}[5m])

# Indexing latency
rate(elasticsearch_indices_indexing_index_time_seconds_total{cluster="$CLUSTER"}[5m])
  / rate(elasticsearch_indices_indexing_index_total{cluster="$CLUSTER"}[5m])

# Index size
elasticsearch_indices_store_size_bytes{cluster="$CLUSTER"}

# Document count
elasticsearch_indices_docs{cluster="$CLUSTER"}
```

**Search Performance**:
```promql
# Search rate
rate(elasticsearch_indices_search_query_total{cluster="$CLUSTER"}[5m])

# Search latency
rate(elasticsearch_indices_search_query_time_seconds_total{cluster="$CLUSTER"}[5m])
  / rate(elasticsearch_indices_search_query_total{cluster="$CLUSTER"}[5m])

# Fetch latency
rate(elasticsearch_indices_search_fetch_time_seconds_total{cluster="$CLUSTER"}[5m])
  / rate(elasticsearch_indices_search_fetch_total{cluster="$CLUSTER"}[5m])
```

**Index Health**:
```promql
# Segment count
elasticsearch_indices_segments_count{cluster="$CLUSTER"}

# Field data size
elasticsearch_indices_fielddata_memory_size_bytes{cluster="$CLUSTER"}

# Query cache size
elasticsearch_indices_query_cache_memory_size_bytes{cluster="$CLUSTER"}
```

### Performance Thresholds

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| Indexing Latency | < 50ms | 50-200ms | > 200ms |
| Search Latency | < 100ms | 100-500ms | > 500ms |
| Fetch Latency | < 50ms | 50-200ms | > 200ms |
| Segment Count | < 50/shard | 50-100 | > 100 |

---

## Phase 5: Query Analysis

### Objective
Identify problematic queries affecting performance.

### Slow Query Analysis

**Enable slow log temporarily** (if needed):
```json
PUT /_settings
{
  "index.search.slowlog.threshold.query.warn": "1s",
  "index.search.slowlog.threshold.query.info": "500ms",
  "index.indexing.slowlog.threshold.index.warn": "1s"
}
```

### Common Problem Patterns

| Pattern | Issue | Fix |
|---------|-------|-----|
| Deep pagination | `from` > 10000 | Use search_after |
| Wildcard prefix | `query: *abc` | Use ngrams or exact match |
| Too many fields | High fielddata | Reduce mappings |
| Large aggregations | Unbounded aggs | Add size limits |
| Missing filters | Full scan | Add filters before aggs |

### Query Categories

```promql
# Query vs Fetch ratio (high = query-heavy)
rate(elasticsearch_indices_search_query_total{cluster="$CLUSTER"}[5m])
  / rate(elasticsearch_indices_search_fetch_total{cluster="$CLUSTER"}[5m])
```

---

## Phase 6: Service Impact

### Objective
Identify services affected by ES issues.

### Find Dependent Services

```sql
SELECT s.service_name, s.service_priority, sem.use_case
FROM services s
JOIN service_es_mapping sem ON s.id = sem.service_id
WHERE sem.cluster_name = '$CLUSTER_NAME';
```

### Service Health Check

```promql
# Service error rates using ES
sum(rate(http_client_requests_seconds_count{app=~"$APPS",uri=~".*elasticsearch.*",status=~"5.."}[5m]))
  / sum(rate(http_client_requests_seconds_count{app=~"$APPS",uri=~".*elasticsearch.*"}[5m]))

# Search service latency
histogram_quantile(0.99, rate(es_search_latency_bucket{service=~"$SERVICES"}[5m]))
```

---

## Phase 7: Report Generation

### Standard Report Template

```markdown
# Elasticsearch Alert Investigation Report

**Report ID**: [Auto-generated]
**Timestamp**: [Investigation time]

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Cluster | [cluster-name] |
| Domain | [AWS domain or self-hosted] |
| Tier | [Search/Logging/Analytics] |
| Alert Type | [Category] |
| Priority | [L0/L1/L2] |

## 2. Cluster Health

| Metric | Current | Status |
|--------|---------|--------|
| Cluster Status | [green/yellow/red] | [OK/WARN/CRIT] |
| Total Nodes | N | [OK/WARN] |
| Data Nodes | M | [OK/WARN] |
| Active Shards | X | - |
| Unassigned Shards | Y | [OK/WARN/CRIT] |
| Pending Tasks | Z | [OK/WARN] |

## 3. Node Health

| Node | Role | JVM Heap | CPU | Disk | GC Time |
|------|------|----------|-----|------|---------|
| [node1] | data | X% | Y% | Z% | Ns |

## 4. Index Performance

| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Indexing Rate | X/s | - | - |
| Indexing Latency | Xms | 50ms | [OK/WARN/CRIT] |
| Search Rate | Y/s | - | - |
| Search Latency | Yms | 100ms | [OK/WARN/CRIT] |

## 5. Problem Indices (if any)

| Index | Size | Docs | Segments | Status |
|-------|------|------|----------|--------|
| [index] | X GB | N | M | [issue] |

## 6. Affected Services

| Service | Priority | Error Rate | Latency P99 | Status |
|---------|----------|------------|-------------|--------|
| [name] | [L0/L1] | X% | Yms | [OK/WARN] |

## 7. Root Cause Analysis

**Primary Cause**: [Determined cause]

**Evidence**:
1. [Evidence with metric reference]
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
| UMBQuerier-Luckin | `df8o21agxtkw0d` | ES metrics |

### Cluster Types

| Type | Use Case | Priority |
|------|----------|----------|
| Search | Product/order search | L0 |
| Logging | Application logs | L1 |
| Metrics | System metrics | L2 |

### Root Cause Quick Reference

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| Red cluster | Data node down | Restart node, check disk |
| Yellow cluster | Replica unassigned | Add capacity or rebalance |
| High JVM | Memory pressure | Increase heap, check queries |
| High latency | Slow queries | Analyze slow log |
| Thread pool rejections | Overloaded | Scale out or throttle |
| Disk pressure | Full disk | Delete old indices, add storage |

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-19 | DevOps | Initial release |
