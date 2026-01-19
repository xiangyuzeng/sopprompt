# EC2 Alert Investigation SOP v5.0

**Version**: 5.0
**Updated**: 2025-01-19
**Scope**: AWS EC2 Instance Alert Diagnosis
**Integration**: MCP with 150+ Data Endpoints

---

## Executive Summary

This Standard Operating Procedure provides a systematic 8-phase methodology for investigating EC2 instance alerts. Version 5.0 introduces parallel query execution, enhanced cross-system correlation, and automated fallback mechanisms.

---

## Table of Contents

1. [Alert Categories & Priorities](#1-alert-categories--priorities)
2. [Phase 0: Alert Validation](#phase-0-alert-validation)
3. [Phase 1: Data Availability Check](#phase-1-data-availability-check)
4. [Phase 2: Cross-System Health Assessment](#phase-2-cross-system-health-assessment)
5. [Phase 3: Sibling Instance Comparison](#phase-3-sibling-instance-comparison)
6. [Phase 4: Service-Specific Diagnostics](#phase-4-service-specific-diagnostics)
7. [Phase 5: Dependency Correlation](#phase-5-dependency-correlation)
8. [Phase 6: CloudWatch Integration](#phase-6-cloudwatch-integration)
9. [Phase 7: Root Cause Determination](#phase-7-root-cause-determination)
10. [Phase 8: Report Generation](#phase-8-report-generation)
11. [Appendix: Data Source Reference](#appendix-data-source-reference)

---

## 1. Alert Categories & Priorities

### Alert Type Matrix

| Category | Metric Pattern | Warning Threshold | Critical Threshold | Default Priority |
|----------|---------------|-------------------|-------------------|------------------|
| **Disk** | `node_filesystem_avail_bytes` | < 20% free | < 10% free | L1 |
| **Memory** | `node_memory_MemAvailable_bytes` | < 15% free | < 10% free | L0 |
| **CPU** | `node_cpu_seconds_total{mode="idle"}` | < 20% idle | < 10% idle | L1 |
| **I/O Wait** | `node_cpu_seconds_total{mode="iowait"}` | > 20% | > 40% | L1 |
| **Network** | `node_network_receive_errs_total` | > 10/s | > 50/s | L2 |
| **Load** | `node_load15` | > cores × 2 | > cores × 4 | L1 |

### Service Priority Classification

| Priority | Response SLA | Example Services | Escalation Path |
|----------|-------------|------------------|-----------------|
| **L0** | 5-15 min | payment, order, auth | Immediate page |
| **L1** | 15-30 min | inventory, user, gateway | Slack + ticket |
| **L2** | 30min-2hr | analytics, batch, logging | Ticket only |

---

## Phase 0: Alert Validation

### Objective
Verify alert accuracy and extract investigation context.

### Critical Validation Steps

1. **Confirm Alert-Metric Match**
   - Alert name may not reflect actual issue
   - Example: "memory_alert" could be triggered by disk issues

2. **Extract Metadata**
   ```
   Instance ID: _______________
   Instance IP: _______________
   Service Name: ______________
   Alert Metric: ______________
   Alert Value: _______________
   Threshold: _________________
   Duration: __________________
   ```

3. **Determine Service Priority**
   ```sql
   SELECT service_name, service_priority, owner_team
   FROM services
   WHERE instance_ip = '$INSTANCE_IP';
   ```

### Validation Checklist
- [ ] Alert name matches metric type
- [ ] Instance exists and is running
- [ ] Service priority determined
- [ ] Owner team identified

---

## Phase 1: Data Availability Check

### Objective
Confirm monitoring coverage and data access.

### Prometheus Connectivity

**Primary Datasource**: UMBQuerier-Luckin (UID: `df8o21agxtkw0d`)

```promql
# Verify node exporter is reporting
up{job="node-exporter", instance=~"$IP.*"}

# Check data freshness (should be < 60s)
time() - node_time_seconds{instance=~"$IP.*"}
```

### Instance Identification

```sql
-- Locate EC2 instance details
SELECT instance_id, instance_ip, instance_type,
       launch_time, availability_zone, service_name
FROM ec2_instances
WHERE instance_ip = '$IP' OR instance_id = '$INSTANCE_ID';
```

### Sibling Instance Discovery

```sql
-- Find peer instances for comparison
SELECT instance_ip, instance_id, instance_type
FROM ec2_instances
WHERE service_name = '$SERVICE_NAME'
  AND status = 'running'
  AND instance_ip != '$IP';
```

### Fallback Procedures

| Failure | Fallback Action |
|---------|----------------|
| Prometheus unavailable | Use CloudWatch metrics exclusively |
| Node exporter missing | Check if instance < 1 hour old |
| DevOps DB timeout | Use cached topology (24h) |

---

## Phase 2: Cross-System Health Assessment

### Objective
Collect comprehensive resource metrics.

### **CRITICAL: Execute These Queries in Parallel**

**Query Set A: Storage & Memory**
```promql
# Disk usage by filesystem
100 - (node_filesystem_avail_bytes{instance=~"$IP.*",fstype!~"tmpfs|overlay"}
       / node_filesystem_size_bytes{instance=~"$IP.*",fstype!~"tmpfs|overlay"} * 100)

# Memory utilization
100 - (node_memory_MemAvailable_bytes{instance=~"$IP.*"}
       / node_memory_MemTotal_bytes{instance=~"$IP.*"} * 100)

# Swap usage
node_memory_SwapTotal_bytes{instance=~"$IP.*"} - node_memory_SwapFree_bytes{instance=~"$IP.*"}
```

**Query Set B: CPU & Load**
```promql
# CPU utilization by mode
sum by(mode)(rate(node_cpu_seconds_total{instance=~"$IP.*"}[5m])) * 100

# Overall CPU usage
100 - (avg(rate(node_cpu_seconds_total{mode="idle",instance=~"$IP.*"}[5m])) * 100)

# System load
node_load1{instance=~"$IP.*"}
node_load5{instance=~"$IP.*"}
node_load15{instance=~"$IP.*"}
```

**Query Set C: I/O & Network**
```promql
# Disk I/O utilization
rate(node_disk_io_time_seconds_total{instance=~"$IP.*"}[5m]) * 100

# Disk read/write throughput
rate(node_disk_read_bytes_total{instance=~"$IP.*"}[5m])
rate(node_disk_written_bytes_total{instance=~"$IP.*"}[5m])

# Network throughput
rate(node_network_receive_bytes_total{instance=~"$IP.*",device!~"lo|docker.*|veth.*"}[5m])
rate(node_network_transmit_bytes_total{instance=~"$IP.*",device!~"lo|docker.*|veth.*"}[5m])

# Network errors
rate(node_network_receive_errs_total{instance=~"$IP.*"}[5m])
rate(node_network_transmit_errs_total{instance=~"$IP.*"}[5m])
```

### Health Status Matrix

| Resource | OK | Warning | Critical |
|----------|-----|---------|----------|
| Disk | < 80% | 80-90% | > 90% |
| Memory | < 85% | 85-95% | > 95% |
| CPU | < 70% | 70-85% | > 85% |
| I/O Util | < 70% | 70-90% | > 90% |
| Load | < cores | < cores×2 | > cores×2 |

---

## Phase 3: Sibling Instance Comparison

### Objective
Identify anomalies by comparing with peer instances.

### Comparison Metrics

For each metric, calculate:
- Group mean (μ)
- Standard deviation (σ)
- Z-score for affected instance: `z = (x - μ) / σ`

**Flag as Anomaly if**: |z| > 1.5 (1.5 standard deviations from mean)

### Prometheus Group Queries

```promql
# Disk usage across service group
100 - (node_filesystem_avail_bytes{instance=~"$SERVICE_IP_PATTERN.*",mountpoint="/"}
       / node_filesystem_size_bytes{instance=~"$SERVICE_IP_PATTERN.*",mountpoint="/"} * 100)

# Memory across service group
100 - (node_memory_MemAvailable_bytes{instance=~"$SERVICE_IP_PATTERN.*"}
       / node_memory_MemTotal_bytes{instance=~"$SERVICE_IP_PATTERN.*"} * 100)

# CPU across service group
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle",instance=~"$SERVICE_IP_PATTERN.*"}[5m])) * 100)
```

### Sibling Risk Assessment

Check if any sibling is approaching thresholds:
- Disk > 80% → Flag for proactive cleanup
- Memory > 85% → Flag for scaling review
- CPU sustained > 70% → Flag for load balancing

**Historical Lesson**: The luckysddladmin incident (40GB unrotated logs) was detected on a sibling at 88% before failure.

---

## Phase 4: Service-Specific Diagnostics

### Java/Spring Boot Services

```promql
# JVM heap utilization
jvm_memory_used_bytes{instance=~"$IP.*",area="heap"}
  / jvm_memory_max_bytes{instance=~"$IP.*",area="heap"} * 100

# GC pause time
rate(jvm_gc_pause_seconds_sum{instance=~"$IP.*"}[5m])

# Thread count
jvm_threads_live_threads{instance=~"$IP.*"}
```

### Database Services (MySQL on EC2)

```promql
# MySQL connections
mysql_global_status_threads_connected{instance=~"$IP.*"}
mysql_global_variables_max_connections{instance=~"$IP.*"}

# Query throughput
rate(mysql_global_status_queries{instance=~"$IP.*"}[5m])
```

### Monitoring/Agent Services

```promql
# Process-level metrics
process_cpu_seconds_total{instance=~"$IP.*"}
process_resident_memory_bytes{instance=~"$IP.*"}
process_open_fds{instance=~"$IP.*"}
```

### File Descriptor Check

```promql
# FD utilization
process_open_fds{instance=~"$IP.*"} / process_max_fds{instance=~"$IP.*"} * 100
```

---

## Phase 5: Dependency Correlation

### Objective
Check health of connected databases and caches.

### Database Dependencies

```sql
-- Find connected databases
SELECT db.host, db.port, db.database_name, db.engine
FROM service_db_mapping sdm
JOIN databases db ON sdm.db_id = db.id
WHERE sdm.service_name = '$SERVICE_NAME';
```

**For each database, check**:
```promql
# Connection pool utilization
mysql_global_status_threads_connected{instance="$DB_HOST:9104"}

# Query latency
rate(mysql_global_status_slow_queries{instance="$DB_HOST:9104"}[5m])
```

### Cache Dependencies

```sql
-- Find connected Redis clusters
SELECT rc.cluster_name, rc.endpoint, rc.port
FROM service_redis_mapping srm
JOIN redis_clusters rc ON srm.redis_id = rc.id
WHERE srm.service_name = '$SERVICE_NAME';
```

**For each Redis cluster, check**:
```promql
# Memory usage
redis_memory_used_bytes{cluster="$CLUSTER_NAME"}
  / redis_memory_max_bytes{cluster="$CLUSTER_NAME"} * 100

# Connection count
redis_connected_clients{cluster="$CLUSTER_NAME"}
```

### Cross-Skill Integration

If dependency issues found:
- Database problems → Suggest `/investigate-rds`
- Cache problems → Suggest `/investigate-redis`

---

## Phase 6: CloudWatch Integration

### Objective
Cross-reference with AWS-native metrics.

### EC2 CloudWatch Metrics

Query these metrics for the instance:
- `CPUUtilization`
- `DiskReadOps`, `DiskWriteOps`
- `NetworkIn`, `NetworkOut`
- `StatusCheckFailed_Instance`
- `StatusCheckFailed_System`

### CloudWatch Logs Insights

```sql
-- Search for errors in system logs
fields @timestamp, @message
| filter @logStream like '$INSTANCE_ID'
| filter @message like /error|fail|critical|oom/i
| sort @timestamp desc
| limit 50
```

### AWS Events Check

- Recent EC2 events (maintenance, scheduled)
- Auto Scaling activities
- EBS volume events

---

## Phase 7: Root Cause Determination

### Symptom-to-Cause Matrix

| Symptom Pattern | Likely Root Cause | Evidence to Collect |
|-----------------|-------------------|---------------------|
| High CPU + Normal Memory | CPU-bound workload | Process list, flame graph |
| High Memory + Swap Active | Memory leak or undersized | Heap dump, memory trend |
| High Disk + Large /var/log | Log rotation failure | File sizes, cron status |
| High I/O Wait + Slow Disk | Storage performance | IOPS, latency metrics |
| Network Errors + Retransmits | Network saturation | Security groups, NACLs |
| All Resources High | Traffic spike | Request rates, LB metrics |

### Resolution Actions

| Root Cause | Immediate Action | Long-term Fix |
|------------|------------------|---------------|
| Disk full | Clear logs/temp | Expand volume, add rotation |
| Memory exhaustion | Restart service | Increase memory, fix leak |
| CPU saturation | Scale out | Optimize code, right-size |
| I/O bottleneck | Reduce load | Upgrade to faster storage |
| Network issues | Check configs | Review architecture |

---

## Phase 8: Report Generation

### Standard Report Template

```markdown
# EC2 Alert Investigation Report

**Report ID**: [Auto-generated]
**Timestamp**: [Investigation time]
**Investigator**: [Name/System]

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Instance | [ID] ([IP]) |
| Service | [Name] |
| Priority | [L0/L1/L2] |
| Alert Type | [Category] |
| Duration | [Time since trigger] |
| Status | [Active/Resolved] |

## 2. Investigation Timeline

| Time | Action | Finding |
|------|--------|---------|
| T+0 | Alert received | [Details] |
| T+X | Phase N completed | [Finding] |

## 3. Resource Status

| Resource | Current | Threshold | Status | Trend |
|----------|---------|-----------|--------|-------|
| Disk (/) | X% | 90% | [OK/WARN/CRIT] | [↑/↓/→] |
| Memory | X% | 90% | [OK/WARN/CRIT] | [↑/↓/→] |
| CPU | X% | 80% | [OK/WARN/CRIT] | [↑/↓/→] |
| I/O | X% | 90% | [OK/WARN/CRIT] | [↑/↓/→] |
| Network | X err/s | 10/s | [OK/WARN/CRIT] | [↑/↓/→] |

## 4. Sibling Comparison

| Instance | Service | Disk | Memory | CPU | Status |
|----------|---------|------|--------|-----|--------|
| [Affected] | [Name] | X% | X% | X% | ALERT |
| [Sibling1] | [Name] | X% | X% | X% | OK |
| [Sibling2] | [Name] | X% | X% | X% | OK |

## 5. Dependency Status

| Dependency | Type | Status | Latency |
|------------|------|--------|---------|
| [DB Name] | MySQL | [OK/WARN] | Xms |
| [Cache Name] | Redis | [OK/WARN] | Xms |

## 6. Root Cause Analysis

**Primary Cause**: [Determined cause]

**Evidence**:
1. [Evidence point 1 with metric/log reference]
2. [Evidence point 2]
3. [Evidence point 3]

**Contributing Factors**:
- [Factor 1]
- [Factor 2]

## 7. Recommendations

| Priority | Action | Owner | Timeline |
|----------|--------|-------|----------|
| **Immediate** | [Action] | [Team] | Now |
| **Short-term** | [Action] | [Team] | This week |
| **Long-term** | [Action] | [Team] | This quarter |

## 8. Related Alerts

| Alert | Time | Instance | Status |
|-------|------|----------|--------|
| [Name] | [Time] | [Instance] | [Status] |

## 9. Attachments

- [ ] Metric screenshots
- [ ] Log excerpts
- [ ] Configuration snippets
```

---

## Appendix: Data Source Reference

### Prometheus Datasources

| Name | UID | Jobs | Purpose |
|------|-----|------|---------|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | 95+ | Primary infrastructure |
| prometheus_redis | `ff6p0gjt24phce` | 74 | Redis clusters |

### MySQL Servers by Domain

| Domain | Server Count | Key Databases |
|--------|--------------|---------------|
| Sales/CRM | 15 | order_db, payment_db |
| SCM | 12 | inventory_db, warehouse_db |
| Operations | 10 | shop_db, delivery_db |
| DevOps | 8 | monitor_db, config_db |
| Finance | 6 | finance_db, billing_db |

### Service Owner Contacts

| Domain | Primary Contact | Escalation |
|--------|-----------------|------------|
| Payment | payment-team@ | CTO |
| Order | order-team@ | VP Eng |
| Infrastructure | devops-team@ | CTO |

### Emergency Procedures

**L0 Service Down > 5 Minutes**:
1. Page on-call immediately
2. Initiate incident bridge
3. Begin parallel investigation and mitigation

**Multiple Instances Affected**:
1. Check for common cause (AZ, deployment)
2. Consider traffic shifting
3. Escalate to architecture team

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 5.0 | 2025-01-19 | DevOps | Parallel queries, cross-skill integration |
| 4.0 | 2025-01-11 | DevOps | Sibling comparison, lessons learned |
| 3.0 | 2024-12-15 | DevOps | CloudWatch integration |
| 2.0 | 2024-11-20 | DevOps | Dependency correlation |
| 1.0 | 2024-10-01 | DevOps | Initial release |
