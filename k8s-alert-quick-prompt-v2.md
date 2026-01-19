# K8s Alert Investigation - Quick Prompt v2.0

**Purpose**: Rapid Kubernetes/EKS alert diagnosis template for Claude Code with MCP integration.

---

## Alert Context

```
Cluster: prod-worker01-eks-us
Namespace: ________________
Pod Name: _________________
Container: ________________
Workload: _________________
Alert Type: OOMKilled / CrashLoopBackOff / Pending / Evicted / Throttling
Service Priority: L0 / L1 / L2
Restart Count: ____________
```

---

## Phase 0: Quick Classification

| Alert Type | Priority | First Action |
|------------|----------|--------------|
| OOMKilled | L0/L1 | Check memory usage vs limit |
| CrashLoopBackOff | L0/L1 | Get previous container logs |
| Pending | L1/L2 | Check scheduling events |
| CPU Throttling | L1/L2 | Check throttling metrics |
| Node NotReady | L0 | Check node status |

---

## Phase 1: EKS MCP Commands

```
# Get pod details
manage_k8s_resource(operation="read", cluster_name="prod-worker01-eks-us",
                    kind="Pod", api_version="v1", name="$POD", namespace="$NS")

# Get pod events
get_k8s_events(cluster_name="prod-worker01-eks-us", kind="Pod",
               name="$POD", namespace="$NS")

# Get container logs
get_pod_logs(cluster_name="prod-worker01-eks-us", namespace="$NS",
             pod_name="$POD", container_name="$CONTAINER", tail_lines=100)

# Get previous logs (for crashes)
get_pod_logs(..., previous=true)
```

---

## Phase 2: Prometheus Queries (Parallel)

### Memory
```promql
container_memory_working_set_bytes{namespace="$NS",pod="$POD",container!="POD"}
kube_pod_container_resource_limits{namespace="$NS",pod="$POD",resource="memory"}
```

### CPU
```promql
rate(container_cpu_usage_seconds_total{namespace="$NS",pod="$POD",container!="POD"}[5m])
kube_pod_container_resource_limits{namespace="$NS",pod="$POD",resource="cpu"}
```

### Status
```promql
kube_pod_container_status_restarts_total{namespace="$NS",pod="$POD"}
kube_pod_container_status_waiting_reason{namespace="$NS",pod="$POD"}
kube_pod_container_status_terminated_reason{namespace="$NS",pod="$POD"}
```

### Throttling
```promql
rate(container_cpu_cfs_throttled_seconds_total{namespace="$NS",pod="$POD"}[5m])
```

---

## Phase 3: Dependency Check

```sql
-- Database dependencies
SELECT db.host, db.database_name FROM service_db_mapping sdm
JOIN databases db ON sdm.db_id = db.id WHERE sdm.service_name = '$SERVICE';

-- Redis dependencies
SELECT rc.cluster_name FROM service_redis_mapping srm
JOIN redis_clusters rc ON srm.redis_id = rc.id WHERE srm.service_name = '$SERVICE';
```

---

## Quick Reference

### Datasources
| Name | UID | Purpose |
|------|-----|---------|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | kube-state-metrics |
| EKS Cluster | prod-worker01-eks-us | K8s API |

### Namespace Priorities
| Priority | Namespaces |
|----------|------------|
| L0 | payment, order, user-auth |
| L1 | inventory, gateway |
| L2 | monitoring, logging |

---

## Alert-Specific Quick Actions

### OOMKilled
1. Check: `container_memory_working_set_bytes` vs limit
2. Get: Previous container logs
3. Action: Increase memory limit or fix leak

### CrashLoopBackOff
1. Get: `get_pod_logs(..., previous=true)`
2. Check: Startup dependencies
3. Action: Fix startup error

### Pending
1. Get: `get_k8s_events(...)`
2. Check: Node resources, selectors, PVC
3. Action: Add nodes or adjust scheduling

### CPU Throttling
1. Check: `container_cpu_cfs_throttled_seconds_total`
2. If > 25%: CPU limit too low
3. Action: Increase CPU limit

---

## Output Format

```markdown
## K8s Investigation Summary

**Cluster**: prod-worker01-eks-us
**Pod**: [namespace]/[pod-name]
**Workload**: [deployment/statefulset]
**Alert**: [Type]
**Priority**: [L0/L1/L2]

### Pod Status
| Container | Status | Restarts | Last State |
|-----------|--------|----------|------------|
| [name] | [status] | [N] | [reason] |

### Resources
| Resource | Current | Limit | Utilization |
|----------|---------|-------|-------------|
| Memory | X Mi | Y Mi | Z% |
| CPU | X m | Y m | Z% |

### Root Cause
[Primary cause with evidence]

### Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
```

---

## Checklist

- [ ] Pod status and events checked
- [ ] Container logs reviewed
- [ ] Resource usage vs limits analyzed
- [ ] Throttling metrics checked (if applicable)
- [ ] Dependencies verified
- [ ] Root cause determined
- [ ] Recommendations provided

---

## Cross-Skill Integration

- Database issues → `/investigate-rds`
- Cache issues → `/investigate-redis`
- Host node issues → `/investigate-ec2`
- Application issues → `/investigate-apm`
