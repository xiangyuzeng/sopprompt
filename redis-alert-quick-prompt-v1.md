# Redis Alert Investigation - Quick Prompt v1.0

**Purpose**: Rapid Redis/ElastiCache alert diagnosis template for Claude Code with MCP integration.

---

## Alert Context

```
Cluster Name: ________________
Node Endpoint: _______________
Node Role: Primary / Replica
Cluster Tier: Session / Business / Supporting
Service Priority: L0 / L1 / L2
Alert Type: __________________
Alert Value: _________________
```

---

## Phase 0: Quick Classification

| Alert Type | Priority | First Action |
|------------|----------|--------------|
| Memory High | L0/L1 | Check used_memory vs maxmemory |
| Evictions | L0/L1 | Check eviction rate and TTL |
| CPU High | L1 | Find slow commands |
| Latency | L1/L2 | Check ops/sec and slow log |
| Connections | L0/L1 | Check client list |
| Replication | L0 | Check master link status |

---

## Phase 1: Direct Redis Commands

```
# Health check
redis_command(server="$CLUSTER", command="PING", args=[])

# Full info
redis_command(server="$CLUSTER", command="INFO", args=["all"])

# Memory analysis
redis_command(server="$CLUSTER", command="MEMORY", args=["STATS"])

# Slow commands
redis_command(server="$CLUSTER", command="SLOWLOG", args=["GET", "20"])

# Client connections
redis_command(server="$CLUSTER", command="CLIENT", args=["LIST"])
```

---

## Phase 2: Prometheus Queries (Parallel)

### Memory
```promql
redis_memory_used_bytes{cluster="$CLUSTER"}
redis_memory_max_bytes{cluster="$CLUSTER"}
redis_memory_fragmentation_ratio{cluster="$CLUSTER"}
rate(redis_evicted_keys_total{cluster="$CLUSTER"}[5m])
```

### Performance
```promql
redis_instantaneous_ops_per_sec{cluster="$CLUSTER"}
redis_connected_clients{cluster="$CLUSTER"}
redis_blocked_clients{cluster="$CLUSTER"}
```

### Network
```promql
rate(redis_net_input_bytes_total{cluster="$CLUSTER"}[5m])
rate(redis_net_output_bytes_total{cluster="$CLUSTER"}[5m])
```

---

## Phase 3: Service Dependencies

```sql
SELECT s.service_name, s.service_priority, srm.cache_purpose
FROM services s
JOIN service_redis_mapping srm ON s.id = srm.service_id
WHERE srm.cluster_name = '$CLUSTER_NAME';
```

---

## Quick Reference

### Datasources
| Name | UID | Purpose |
|------|-----|---------|
| Redis | mcp-db-gateway | Direct commands |
| Prometheus | `ff6p0gjt24phce` | Metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Topology |

### Thresholds
| Metric | Warning | Critical |
|--------|---------|----------|
| Memory | 75% | 85% |
| Fragmentation | 1.3 | 1.5 |
| Evictions/min | 100 | 500 |
| Blocked Clients | 5 | 10 |

### Cluster Tiers
| Tier | Examples | Priority |
|------|----------|----------|
| Session | session-cache, auth-cache | L0 |
| Business | product-cache, cart-cache | L1 |
| Supporting | config-cache, analytics-cache | L2 |

---

## Alert-Specific Quick Actions

### Memory High
1. Check: `redis_memory_used_bytes` vs `redis_memory_max_bytes`
2. Get: `MEMORY STATS` for breakdown
3. Action: Check TTL policy, eviction policy

### Evictions
1. Check: `rate(redis_evicted_keys_total[5m])`
2. Get: `INFO keyspace` for key distribution
3. Action: Increase memory or adjust TTL

### High CPU / Latency
1. Get: `SLOWLOG GET 20`
2. Look for: O(N) operations (KEYS, SMEMBERS on large sets)
3. Action: Optimize commands, use SCAN

### Connection Issues
1. Get: `CLIENT LIST`
2. Check: Connection distribution, idle connections
3. Action: Tune connection pools

---

## Output Format

```markdown
## Redis Investigation Summary

**Cluster**: [cluster-name]
**Node**: [Primary/Replica] [endpoint]
**Tier**: [Session/Business/Supporting]
**Alert**: [Type] - [Value] vs [Threshold]

### Health Status
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Memory | X% | 85% | [OK/WARN/CRIT] |
| Fragmentation | X.XX | 1.5 | [OK/WARN/CRIT] |
| Evictions/min | N | 100 | [OK/WARN/CRIT] |
| Ops/sec | N | - | - |

### Slow Commands (if any)
| Command | Duration | Key Pattern |
|---------|----------|-------------|

### Root Cause
[Primary cause with evidence]

### Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
```

---

## Checklist

- [ ] Redis connectivity verified
- [ ] Memory usage checked
- [ ] Slow log reviewed
- [ ] Client connections analyzed
- [ ] Eviction rate checked
- [ ] Dependent services reviewed
- [ ] Root cause determined

---

## Cross-Skill Integration

- Database issues -> `/investigate-rds`
- Application issues -> `/investigate-apm`
- K8s pod issues -> `/investigate-k8s`
- Host issues -> `/investigate-ec2`
