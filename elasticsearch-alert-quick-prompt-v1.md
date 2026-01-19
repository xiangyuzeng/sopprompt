# Elasticsearch Alert Investigation - Quick Prompt v1.0

**Purpose**: Rapid Elasticsearch/OpenSearch alert diagnosis template for Claude Code with MCP integration.

---

## Alert Context

```
Cluster: ____________________
Endpoint: ___________________
Node Count: _________________
Cluster Tier: Search / Logging / Analytics
Service Priority: L0 / L1 / L2
Alert Type: __________________
Alert Value: _________________
```

---

## Phase 0: Quick Classification

| Alert Type | Priority | First Action |
|------------|----------|--------------|
| Cluster Red | L0 | Check data node status |
| Cluster Yellow | L0/L1 | Check unassigned shards |
| JVM Heap High | L0/L1 | Check GC metrics |
| CPU High | L1 | Find hot threads |
| Disk High | L0/L1 | Check large indices |
| Search Latency | L1/L2 | Analyze slow queries |

---

## Phase 1: Cluster Health (Parallel)

```promql
# Cluster status
elasticsearch_cluster_health_status{cluster="$CLUSTER"}

# Node count
elasticsearch_cluster_health_number_of_nodes{cluster="$CLUSTER"}
elasticsearch_cluster_health_number_of_data_nodes{cluster="$CLUSTER"}

# Shard allocation
elasticsearch_cluster_health_active_shards{cluster="$CLUSTER"}
elasticsearch_cluster_health_unassigned_shards{cluster="$CLUSTER"}
elasticsearch_cluster_health_relocating_shards{cluster="$CLUSTER"}
```

---

## Phase 2: Node Metrics (Parallel)

```promql
# JVM heap per node
elasticsearch_jvm_memory_used_bytes{cluster="$CLUSTER",area="heap"}
  / elasticsearch_jvm_memory_max_bytes{cluster="$CLUSTER",area="heap"} * 100

# GC time
rate(elasticsearch_jvm_gc_collection_seconds_sum{cluster="$CLUSTER"}[5m])

# CPU utilization
elasticsearch_os_cpu_percent{cluster="$CLUSTER"}

# Disk usage
(elasticsearch_filesystem_data_size_bytes{cluster="$CLUSTER"}
  - elasticsearch_filesystem_data_free_bytes{cluster="$CLUSTER"})
  / elasticsearch_filesystem_data_size_bytes{cluster="$CLUSTER"} * 100
```

---

## Phase 3: Performance Metrics

```promql
# Indexing latency
rate(elasticsearch_indices_indexing_index_time_seconds_total{cluster="$CLUSTER"}[5m])
  / rate(elasticsearch_indices_indexing_index_total{cluster="$CLUSTER"}[5m])

# Search latency
rate(elasticsearch_indices_search_query_time_seconds_total{cluster="$CLUSTER"}[5m])
  / rate(elasticsearch_indices_search_query_total{cluster="$CLUSTER"}[5m])

# Thread pool rejections
rate(elasticsearch_thread_pool_rejected_count{cluster="$CLUSTER"}[5m])
```

---

## Phase 4: Service Dependencies

```sql
SELECT s.service_name, s.service_priority, sem.use_case
FROM services s
JOIN service_es_mapping sem ON s.id = sem.service_id
WHERE sem.cluster_name = '$CLUSTER_NAME';
```

---

## Quick Reference

### Datasources
| Name | UID | Purpose |
|------|-----|---------|
| Prometheus | `df8o21agxtkw0d` | ES metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Topology |

### Thresholds
| Metric | Warning | Critical |
|--------|---------|----------|
| JVM Heap | 75% | 85% |
| CPU | 70% | 85% |
| Disk | 75% | 85% |
| Search Latency | 100ms | 500ms |
| Index Latency | 50ms | 200ms |

### Cluster Status
| Status | Meaning |
|--------|---------|
| Green | All shards OK |
| Yellow | Missing replicas |
| Red | Missing primaries |

---

## Alert-Specific Quick Actions

### Cluster Red
1. Check: Data node count vs expected
2. Get: Unassigned shard reasons
3. Action: Recover/restart affected nodes

### Cluster Yellow
1. Check: `elasticsearch_cluster_health_unassigned_shards`
2. Verify: Node capacity for replicas
3. Action: Add nodes or reduce replicas

### JVM Heap High
1. Check: GC frequency and time
2. Look for: Memory-heavy queries
3. Action: Increase heap or optimize queries

### High Search Latency
1. Check: Search rate vs latency trend
2. Look for: Expensive queries
3. Action: Optimize queries, add caching

### Disk Pressure
1. Check: Index sizes by age
2. Find: Large/old indices
3. Action: Delete old data or add storage

---

## Output Format

```markdown
## Elasticsearch Investigation Summary

**Cluster**: [cluster-name]
**Status**: [green/yellow/red]
**Nodes**: [X data / Y total]
**Alert**: [Type] - [Value] vs [Threshold]

### Cluster Health
| Metric | Current | Status |
|--------|---------|--------|
| Status | [color] | [OK/WARN/CRIT] |
| Data Nodes | N | [OK/WARN] |
| Unassigned Shards | X | [OK/WARN/CRIT] |

### Node Health
| Node | JVM Heap | CPU | Disk |
|------|----------|-----|------|
| [node] | X% | Y% | Z% |

### Performance
| Metric | P50 | P99 | Status |
|--------|-----|-----|--------|
| Search Latency | Xms | Yms | [OK/WARN] |
| Index Latency | Xms | Yms | [OK/WARN] |

### Root Cause
[Primary cause with evidence]

### Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
```

---

## Checklist

- [ ] Cluster health verified
- [ ] Node status checked
- [ ] JVM/GC metrics reviewed
- [ ] Index performance analyzed
- [ ] Disk usage verified
- [ ] Dependent services reviewed
- [ ] Root cause determined

---

## Cross-Skill Integration

- Cache issues -> `/investigate-redis`
- Database issues -> `/investigate-rds`
- Application issues -> `/investigate-apm`
- K8s pod issues -> `/investigate-k8s`
