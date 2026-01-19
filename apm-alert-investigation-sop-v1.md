# APM Alert Investigation SOP v1.0

**Version**: 1.0
**Updated**: 2025-01-19
**Scope**: Application Performance Monitoring Alert Diagnosis
**Integration**: MCP with Prometheus, Grafana, CloudWatch

---

## Executive Summary

This Standard Operating Procedure provides a systematic 8-phase methodology for investigating application performance alerts including latency spikes, error rate increases, throughput degradation, and resource exhaustion.

---

## Table of Contents

1. [Alert Categories & Priorities](#1-alert-categories--priorities)
2. [Phase 0: Alert Validation](#phase-0-alert-validation)
3. [Phase 1: Data Availability Check](#phase-1-data-availability-check)
4. [Phase 2: Service Overview](#phase-2-service-overview)
5. [Phase 3: Latency Analysis](#phase-3-latency-analysis)
6. [Phase 4: Error Analysis](#phase-4-error-analysis)
7. [Phase 5: Throughput Analysis](#phase-5-throughput-analysis)
8. [Phase 6: Dependency Analysis](#phase-6-dependency-analysis)
9. [Phase 7: Resource Correlation](#phase-7-resource-correlation)
10. [Phase 8: Report Generation](#phase-8-report-generation)
11. [Appendix: Data Source Reference](#appendix-data-source-reference)

---

## 1. Alert Categories & Priorities

### Alert Type Matrix

| Category | Metric | Warning | Critical | Default Priority |
|----------|--------|---------|----------|------------------|
| **Latency P99** | http_request_duration_p99 | > 500ms | > 2s | L0/L1 |
| **Latency P50** | http_request_duration_p50 | > 200ms | > 500ms | L1 |
| **Error Rate** | error_rate_5xx | > 1% | > 5% | L0/L1 |
| **Error Rate 4xx** | error_rate_4xx | > 5% | > 15% | L1/L2 |
| **Throughput Drop** | requests_per_second | < 50% baseline | < 25% baseline | L0/L1 |
| **Throughput Spike** | requests_per_second | > 200% baseline | > 500% baseline | L1/L2 |
| **Saturation** | thread_pool_utilization | > 70% | > 85% | L1 |
| **Apdex** | apdex_score | < 0.9 | < 0.7 | L1/L2 |

### Service Priority Classification

| Priority | Response SLA | Services |
|----------|-------------|----------|
| **L0** | 5-15 min | payment-service, order-service, auth-service |
| **L1** | 15-30 min | inventory-service, cart-service, gateway |
| **L2** | 30min-2hr | notification-service, analytics-service |

---

## Phase 0: Alert Validation

### Objective
Verify alert accuracy and extract investigation context.

### Metadata Extraction

```
Service Name: ________________
Service Version: _____________
Environment: prod / staging
Deployment: K8s namespace/deployment
Instance Count: ______________
Service Priority: L0 / L1 / L2
Alert Type: __________________
Alert Value: _________________
Threshold: ___________________
Duration: ____________________
```

### Service Priority Lookup

```sql
SELECT service_name, service_priority, owner_team, slack_channel,
       k8s_namespace, k8s_deployment, instance_count
FROM services
WHERE service_name = '$SERVICE_NAME';
```

---

## Phase 1: Data Availability Check

### Prometheus Connectivity

**Primary Datasource**: UMBQuerier-Luckin (UID: `df8o21agxtkw0d`)

```promql
# Verify service metrics available
up{job="$SERVICE_JOB"}

# Check scrape freshness
scrape_duration_seconds{job="$SERVICE_JOB"}
```

### Validate Metric Coverage

```promql
# HTTP metrics available
http_server_requests_seconds_count{service="$SERVICE"}

# JVM metrics (if Java)
jvm_memory_used_bytes{service="$SERVICE"}

# Custom business metrics
{service="$SERVICE", __name__=~".*business.*"}
```

---

## Phase 2: Service Overview

### Objective
Get overall service health snapshot.

### RED Metrics (Rate, Error, Duration) - Parallel Execution

```promql
# Request Rate (R)
sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# Error Rate (E)
sum(rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# Duration P50 (D)
histogram_quantile(0.5, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))

# Duration P99 (D)
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))
```

### Instance Health Check

```promql
# Per-instance request rate
sum by(instance) (rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# Per-instance error rate
sum by(instance) (rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
  / sum by(instance) (rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
```

### Health Status Matrix

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Request Rate | Stable | ±30% | ±50% |
| Error Rate 5xx | < 0.1% | 0.1-1% | > 1% |
| Latency P99 | < 500ms | 500ms-2s | > 2s |
| Apdex | > 0.95 | 0.85-0.95 | < 0.85 |

---

## Phase 3: Latency Analysis

### Objective
Identify latency degradation sources.

### Overall Latency Percentiles (Parallel)

```promql
# P50, P90, P95, P99 latency
histogram_quantile(0.5, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))
histogram_quantile(0.9, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))
```

### Latency by Endpoint

```promql
# P99 by URI
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le, uri)
)
```

### Latency by Instance

```promql
# P99 per instance (detect slow instances)
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le, instance)
)
```

### Latency Trend Analysis

```promql
# Latency over time (1h window)
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m])) by (le))[1h:1m]
```

### Latency Investigation Matrix

| Pattern | Likely Cause | Action |
|---------|--------------|--------|
| All endpoints slow | Database/cache | Check dependencies |
| Single endpoint slow | Business logic | Profile endpoint |
| Single instance slow | Instance issue | Check instance resources |
| Gradual increase | Memory leak/GC | Check JVM metrics |
| Sudden spike | Deployment/traffic | Check deployments, traffic |

---

## Phase 4: Error Analysis

### Objective
Identify and categorize errors.

### Error Rate by Type (Parallel)

```promql
# 5xx error rate
sum(rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# 4xx error rate
sum(rate(http_server_requests_seconds_count{service="$SERVICE",status=~"4.."}[5m]))
  / sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# Error count by status code
sum by(status) (rate(http_server_requests_seconds_count{service="$SERVICE",status!~"2.."}[5m]))
```

### Error Rate by Endpoint

```promql
# Top error endpoints
topk(10,
  sum by(uri) (rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
)
```

### Error Rate by Instance

```promql
# Per-instance error rate (find problematic instances)
sum by(instance) (rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
  / sum by(instance) (rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
```

### Error Code Analysis

| Status | Meaning | Common Cause |
|--------|---------|--------------|
| 500 | Internal Error | Application bug, unhandled exception |
| 502 | Bad Gateway | Upstream failure, timeout |
| 503 | Service Unavailable | Overloaded, circuit breaker |
| 504 | Gateway Timeout | Slow dependency, timeout config |
| 429 | Too Many Requests | Rate limiting |

---

## Phase 5: Throughput Analysis

### Objective
Analyze request volume patterns.

### Throughput Metrics (Parallel)

```promql
# Overall request rate
sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# Request rate by endpoint
sum by(uri) (rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# Request rate by method
sum by(method) (rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
```

### Throughput Comparison

```promql
# Current vs 1h ago
sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
  / sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m] offset 1h))

# Current vs 1d ago (same time)
sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))
  / sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m] offset 1d))
```

### Throughput Pattern Analysis

| Pattern | Possible Cause | Action |
|---------|----------------|--------|
| Sudden drop | Upstream issue, routing | Check load balancer, upstream |
| Gradual drop | Degraded UX, errors | Check client-side errors |
| Sudden spike | Traffic burst, attack | Check sources, scale if needed |
| Periodic pattern | Scheduled jobs, cron | Verify expected behavior |

---

## Phase 6: Dependency Analysis

### Objective
Check health of downstream dependencies.

### Database Dependencies

```sql
SELECT db.host, db.database_name, sdm.connection_pool_size
FROM service_db_mapping sdm
JOIN databases db ON sdm.db_id = db.id
WHERE sdm.service_name = '$SERVICE_NAME';
```

### HTTP Client Metrics (Parallel)

```promql
# Outbound request latency
histogram_quantile(0.99, sum(rate(http_client_requests_seconds_bucket{app="$SERVICE"}[5m])) by (le, uri))

# Outbound error rate
sum by(uri) (rate(http_client_requests_seconds_count{app="$SERVICE",status=~"5.."}[5m]))
  / sum by(uri) (rate(http_client_requests_seconds_count{app="$SERVICE"}[5m]))
```

### Database Client Metrics

```promql
# Database query latency
histogram_quantile(0.99, sum(rate(db_query_duration_seconds_bucket{app="$SERVICE"}[5m])) by (le))

# Database connection pool usage
db_pool_active_connections{app="$SERVICE"} / db_pool_max_connections{app="$SERVICE"}

# Database errors
rate(db_query_errors_total{app="$SERVICE"}[5m])
```

### Cache Client Metrics

```promql
# Cache latency
histogram_quantile(0.99, sum(rate(cache_operation_duration_seconds_bucket{app="$SERVICE"}[5m])) by (le))

# Cache hit rate
rate(cache_hits_total{app="$SERVICE"}[5m])
  / (rate(cache_hits_total{app="$SERVICE"}[5m]) + rate(cache_misses_total{app="$SERVICE"}[5m]))
```

### Dependency Health Matrix

| Dependency | Latency P99 | Error Rate | Status |
|------------|-------------|------------|--------|
| database | Xms | Y% | [OK/WARN/CRIT] |
| redis | Xms | Y% | [OK/WARN/CRIT] |
| upstream-api | Xms | Y% | [OK/WARN/CRIT] |

---

## Phase 7: Resource Correlation

### Objective
Correlate with infrastructure resources.

### JVM Metrics (Java Services)

```promql
# Heap usage
jvm_memory_used_bytes{service="$SERVICE",area="heap"}
  / jvm_memory_max_bytes{service="$SERVICE",area="heap"} * 100

# GC pause time
rate(jvm_gc_pause_seconds_sum{service="$SERVICE"}[5m])

# Thread count
jvm_threads_live_threads{service="$SERVICE"}
jvm_threads_peak_threads{service="$SERVICE"}
```

### Container Resources

```promql
# CPU usage
rate(container_cpu_usage_seconds_total{namespace="$NS",pod=~"$SERVICE.*"}[5m])

# Memory usage
container_memory_working_set_bytes{namespace="$NS",pod=~"$SERVICE.*"}

# CPU throttling
rate(container_cpu_cfs_throttled_seconds_total{namespace="$NS",pod=~"$SERVICE.*"}[5m])
```

### Connection Pool Status

```promql
# Active connections vs max
hikaricp_connections_active{service="$SERVICE"} / hikaricp_connections_max{service="$SERVICE"}

# Pending connections
hikaricp_connections_pending{service="$SERVICE"}

# Connection timeout rate
rate(hikaricp_connections_timeout_total{service="$SERVICE"}[5m])
```

---

## Phase 8: Report Generation

### Standard Report Template

```markdown
# APM Alert Investigation Report

**Report ID**: [Auto-generated]
**Timestamp**: [Investigation time]

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Service | [service-name] |
| Version | [version] |
| Environment | [prod/staging] |
| Instances | [N] |
| Priority | [L0/L1/L2] |
| Alert Type | [Latency/Error/Throughput] |

## 2. Service Health (RED Metrics)

| Metric | Current | Baseline | Change | Status |
|--------|---------|----------|--------|--------|
| Request Rate | X/s | Y/s | ±Z% | [OK/WARN/CRIT] |
| Error Rate | X% | Y% | ±Z% | [OK/WARN/CRIT] |
| Latency P50 | Xms | Yms | ±Z% | [OK/WARN/CRIT] |
| Latency P99 | Xms | Yms | ±Z% | [OK/WARN/CRIT] |

## 3. Latency Analysis

### By Endpoint (Top 5 Slowest)
| Endpoint | P50 | P99 | Rate | Status |
|----------|-----|-----|------|--------|
| /api/X | Xms | Yms | N/s | [OK/WARN] |

### By Instance
| Instance | P99 | Error Rate | Status |
|----------|-----|------------|--------|
| [ip] | Xms | Y% | [OK/WARN] |

## 4. Error Analysis

### By Status Code
| Status | Rate | % of Errors | Trend |
|--------|------|-------------|-------|
| 500 | X/s | Y% | [up/down/stable] |

### Top Error Endpoints
| Endpoint | Error Rate | Count |
|----------|------------|-------|
| /api/X | Y% | N |

## 5. Dependency Health

| Dependency | Type | Latency P99 | Error Rate | Status |
|------------|------|-------------|------------|--------|
| [db-name] | MySQL | Xms | Y% | [OK/WARN] |
| [cache-name] | Redis | Xms | Y% | [OK/WARN] |
| [api-name] | HTTP | Xms | Y% | [OK/WARN] |

## 6. Resource Utilization

| Resource | Current | Limit | Utilization |
|----------|---------|-------|-------------|
| JVM Heap | X MB | Y MB | Z% |
| CPU | X cores | Y cores | Z% |
| Memory | X MB | Y MB | Z% |
| Threads | N | M max | [OK/WARN] |

## 7. Root Cause Analysis

**Primary Cause**: [Determined cause]

**Evidence**:
1. [Evidence with metric reference]
2. [Evidence]

**Timeline**:
- [Time]: [Event]
- [Time]: [Event]

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
| UMBQuerier-Luckin | `df8o21agxtkw0d` | Service metrics |

### Key Metric Names

| Metric | Purpose |
|--------|---------|
| http_server_requests_seconds | HTTP server latency |
| http_client_requests_seconds | HTTP client latency |
| jvm_memory_used_bytes | JVM memory |
| jvm_gc_pause_seconds | GC pause time |
| hikaricp_connections | DB connection pool |

### Service Owner Contacts

| Service Tier | Contact | Escalation |
|--------------|---------|------------|
| L0 (Critical) | on-call@ | CTO |
| L1 (Important) | team-lead@ | VP Eng |
| L2 (Standard) | team@ | Manager |

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-19 | DevOps | Initial release |
