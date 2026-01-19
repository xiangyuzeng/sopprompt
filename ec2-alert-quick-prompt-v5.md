# EC2 Alert Investigation - Quick Prompt v5.0

**Purpose**: Rapid EC2 alert diagnosis template for Claude Code with MCP integration.

---

## Alert Context

```
Instance ID: _______________
Instance IP: _______________
Service Name: ______________
Service Priority: L0 / L1 / L2
Alert Type: ________________
Alert Value: _______________
Threshold: _________________
Duration: __________________
```

---

## Phase 0: Validation (30s)

**Verify alert matches actual issue**:
```promql
# Check actual metric value
node_filesystem_avail_bytes{instance=~"$IP.*"} / node_filesystem_size_bytes{instance=~"$IP.*"} * 100
node_memory_MemAvailable_bytes{instance=~"$IP.*"} / node_memory_MemTotal_bytes{instance=~"$IP.*"} * 100
100 - avg(rate(node_cpu_seconds_total{mode="idle",instance=~"$IP.*"}[5m])) * 100
```

---

## Phase 1: Parallel Health Check

**Execute ALL queries in parallel**:

### Query Set A: Storage & Memory
```promql
100 - (node_filesystem_avail_bytes{instance=~"$IP.*",fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{instance=~"$IP.*",fstype!~"tmpfs|overlay"} * 100)
100 - (node_memory_MemAvailable_bytes{instance=~"$IP.*"} / node_memory_MemTotal_bytes{instance=~"$IP.*"} * 100)
```

### Query Set B: CPU & Load
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle",instance=~"$IP.*"}[5m])) * 100)
node_load15{instance=~"$IP.*"}
```

### Query Set C: I/O & Network
```promql
rate(node_disk_io_time_seconds_total{instance=~"$IP.*"}[5m]) * 100
rate(node_network_receive_bytes_total{instance=~"$IP.*"}[5m])
rate(node_network_receive_errs_total{instance=~"$IP.*"}[5m])
```

---

## Phase 2: Sibling Comparison

```sql
-- Get sibling instances
SELECT instance_ip, instance_id FROM ec2_instances
WHERE service_name = '$SERVICE' AND status = 'running';
```

**Compare metrics across siblings** - Flag if affected instance > 1.5σ from group mean.

---

## Phase 3: Dependency Check

```sql
-- Database dependencies
SELECT db.host, db.database_name FROM service_db_mapping sdm
JOIN databases db ON sdm.db_id = db.id WHERE sdm.service_name = '$SERVICE';

-- Redis dependencies
SELECT rc.cluster_name, rc.endpoint FROM service_redis_mapping srm
JOIN redis_clusters rc ON srm.redis_id = rc.id WHERE srm.service_name = '$SERVICE';
```

---

## Phase 4: CloudWatch Cross-Reference

- CPUUtilization
- StatusCheckFailed_Instance
- StatusCheckFailed_System
- DiskReadOps, DiskWriteOps

---

## Quick Reference

### Datasources
| Name | UID | Purpose |
|------|-----|---------|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | Primary Prometheus |
| DevOps DB | `aws-luckyus-devops-rw` | Topology |

### Thresholds
| Resource | Warning | Critical |
|----------|---------|----------|
| Disk | 80% | 90% |
| Memory | 85% | 95% |
| CPU | 70% | 85% |
| I/O | 70% | 90% |

### Service Levels
| Priority | Response | Services |
|----------|----------|----------|
| L0 | 5-15 min | payment, order, auth |
| L1 | 15-30 min | inventory, gateway |
| L2 | 30min-2hr | analytics, batch |

---

## Output Format

```markdown
## EC2 Investigation Summary

**Instance**: [ID] ([IP])
**Service**: [Name] (Priority: [L0/L1/L2])
**Alert**: [Type] - [Current Value] vs [Threshold]

### Resource Status
| Resource | Current | Threshold | Status |
|----------|---------|-----------|--------|
| Disk | X% | 90% | [STATUS] |
| Memory | X% | 90% | [STATUS] |
| CPU | X% | 80% | [STATUS] |

### Sibling Comparison
| Instance | Status | Key Metric |
|----------|--------|------------|
| [affected] | ALERT | X% |
| [sibling1] | OK | Y% |

### Root Cause
[Primary cause with evidence]

### Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
3. **Long-term**: [Action]
```

---

## Checklist

- [ ] Alert validated (name matches metric)
- [ ] All resources checked (disk, memory, CPU, I/O, network)
- [ ] Siblings compared
- [ ] Dependencies verified
- [ ] CloudWatch cross-referenced
- [ ] Root cause determined
- [ ] Recommendations provided

---

## Cross-Skill Integration

If issues found in other domains:
- Database issues → `/investigate-rds`
- Cache issues → `/investigate-redis`
- K8s pod issues → `/investigate-k8s`
- Application issues → `/investigate-apm`
