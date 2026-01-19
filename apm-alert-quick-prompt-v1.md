# APM Alert Investigation - Quick Prompt v1.0

**Purpose**: Rapid Application Performance Monitoring alert diagnosis template for Claude Code with MCP integration.

---

## Alert Context

```
Service Name: ________________
Service Version: _____________
Environment: prod / staging
K8s Namespace: _______________
Instance Count: ______________
Service Priority: L0 / L1 / L2
Alert Type: __________________
Alert Value: _________________
```

---

## Phase 0: Quick Classification

| Alert Type | Priority | First Action |
|------------|----------|--------------|
| Latency P99 High | L0/L1 | Check by endpoint and instance |
| Error Rate High | L0/L1 | Check error by status code |
| Throughput Drop | L0/L1 | Check upstream/load balancer |
| Throughput Spike | L1/L2 | Check traffic source |
| Resource Exhaustion | L1 | Check JVM/container metrics |

---

## Phase 1: RED Metrics (Execute in Parallel)

### Rate
```promql
sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
```

### Error
```promql
sum(rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
```

### Duration
```promql
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))
histogram_quantile(0.5, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))
```

---

## Phase 2: Breakdown Analysis (Parallel)

### By Endpoint
```promql
# Slowest endpoints
topk(5, histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le, uri)))

# Highest error endpoints
topk(5, sum by(uri) (rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m])))
```

### By Instance
```promql
# Latency per instance
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le, instance))

# Error rate per instance
sum by(instance) (rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
  / sum by(instance) (rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
```

---

## Phase 3: Dependencies (Parallel)

### Outbound HTTP
```promql
histogram_quantile(0.99, sum(rate(http_client_requests_seconds_bucket{app="$SERVICE"}[5m])) by (le, uri))
sum by(uri) (rate(http_client_requests_seconds_count{app="$SERVICE",status=~"5.."}[5m]))
```

### Database
```promql
histogram_quantile(0.99, sum(rate(db_query_duration_seconds_bucket{app="$SERVICE"}[5m])) by (le))
db_pool_active_connections{app="$SERVICE"} / db_pool_max_connections{app="$SERVICE"}
```

### Cache
```promql
rate(cache_hits_total{app="$SERVICE"}[5m]) / (rate(cache_hits_total{app="$SERVICE"}[5m]) + rate(cache_misses_total{app="$SERVICE"}[5m]))
```

---

## Phase 4: Resources

### JVM (Java services)
```promql
jvm_memory_used_bytes{service="$SERVICE",area="heap"} / jvm_memory_max_bytes{service="$SERVICE",area="heap"} * 100
rate(jvm_gc_pause_seconds_sum{service="$SERVICE"}[5m])
```

### Container
```promql
rate(container_cpu_usage_seconds_total{namespace="$NS",pod=~"$SERVICE.*"}[5m])
container_memory_working_set_bytes{namespace="$NS",pod=~"$SERVICE.*"}
```

---

## Phase 5: Service Dependencies

```sql
-- Find database dependencies
SELECT db.host, db.database_name FROM service_db_mapping sdm
JOIN databases db ON sdm.db_id = db.id WHERE sdm.service_name = '$SERVICE';

-- Find Redis dependencies
SELECT rc.cluster_name FROM service_redis_mapping srm
JOIN redis_clusters rc ON srm.redis_id = rc.id WHERE srm.service_name = '$SERVICE';
```

---

## Quick Reference

### Datasources
| Name | UID | Purpose |
|------|-----|---------|
| Prometheus | `df8o21agxtkw0d` | Service metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Topology |

### Thresholds
| Metric | Warning | Critical |
|--------|---------|----------|
| Latency P99 | 500ms | 2s |
| Error Rate 5xx | 1% | 5% |
| Throughput Change | ±30% | ±50% |
| JVM Heap | 75% | 85% |

### Service Tiers
| Tier | Examples | SLA |
|------|----------|-----|
| L0 | payment, order, auth | 5-15min |
| L1 | inventory, cart, gateway | 15-30min |
| L2 | notification, analytics | 30min-2hr |

---

## Alert-Specific Quick Actions

### Latency P99 High
1. Check: Latency by endpoint and instance
2. Find: Slow dependency (DB/cache/API)
3. Action: Optimize query or scale

### Error Rate High
1. Check: Errors by status code and endpoint
2. Find: Common error pattern
3. Action: Fix bug or dependency issue

### Throughput Drop
1. Check: Upstream service health
2. Verify: Load balancer routing
3. Action: Fix upstream or routing

### Resource Exhaustion
1. Check: JVM heap, GC, threads
2. Find: Memory leak or thread deadlock
3. Action: Restart or fix leak

---

## Output Format

```markdown
## APM Investigation Summary

**Service**: [service-name] v[version]
**Environment**: [prod/staging]
**Instances**: [N]
**Priority**: [L0/L1/L2]
**Alert**: [Type] - [Value] vs [Threshold]

### RED Metrics
| Metric | Current | Baseline | Change | Status |
|--------|---------|----------|--------|--------|
| Rate | X/s | Y/s | ±Z% | [OK/WARN] |
| Error | X% | Y% | ±Z% | [OK/WARN/CRIT] |
| P99 | Xms | Yms | ±Z% | [OK/WARN/CRIT] |

### Slowest Endpoints
| Endpoint | P99 | Rate | Errors |
|----------|-----|------|--------|

### Dependency Health
| Dependency | Latency | Errors | Status |
|------------|---------|--------|--------|

### Root Cause
[Primary cause with evidence]

### Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
```

---

## Checklist

- [ ] RED metrics checked
- [ ] Latency by endpoint analyzed
- [ ] Error distribution reviewed
- [ ] Instance health verified
- [ ] Dependencies checked (DB, cache, APIs)
- [ ] JVM/container resources reviewed
- [ ] Root cause determined

---

## Cross-Skill Integration

- Database issues -> `/investigate-rds`
- Cache issues -> `/investigate-redis`
- K8s pod issues -> `/investigate-k8s`
- Search issues -> `/investigate-elasticsearch`
- Host issues -> `/investigate-ec2`
