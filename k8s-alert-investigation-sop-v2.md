# Kubernetes/EKS Alert Investigation SOP v2.0

**Version**: 2.0
**Updated**: 2025-01-19
**Scope**: Amazon EKS Cluster Alert Diagnosis
**Integration**: MCP with EKS Server, Prometheus, CloudWatch

---

## Executive Summary

This Standard Operating Procedure provides a systematic 9-phase methodology for investigating Kubernetes/EKS alerts. Version 2.0 introduces EKS MCP tool integration, alert-specific playbooks, and enhanced cross-system correlation.

---

## Table of Contents

1. [Alert Categories & Priorities](#1-alert-categories--priorities)
2. [Phase 0: Alert Validation](#phase-0-alert-validation)
3. [Phase 1: Cluster & Node Health](#phase-1-cluster--node-health)
4. [Phase 2: Pod Investigation](#phase-2-pod-investigation)
5. [Phase 3: Resource Analysis](#phase-3-resource-analysis)
6. [Phase 4: Workload Configuration](#phase-4-workload-configuration)
7. [Phase 5: Application Logs](#phase-5-application-logs)
8. [Phase 6: Dependency Analysis](#phase-6-dependency-analysis)
9. [Phase 7: Alert Correlation](#phase-7-alert-correlation)
10. [Phase 8: Alert-Specific Playbooks](#phase-8-alert-specific-playbooks)
11. [Phase 9: Report Generation](#phase-9-report-generation)
12. [Appendix: Data Source Reference](#appendix-data-source-reference)

---

## 1. Alert Categories & Priorities

### Alert Type Matrix

| Category | Trigger Condition | Default Priority | Common Cause |
|----------|-------------------|------------------|--------------|
| **OOMKilled** | Container memory limit exceeded | L0/L1 | Memory leak, undersized limit |
| **CrashLoopBackOff** | Restarts > 5 in 10min | L0/L1 | Startup failure, dependency down |
| **Pending** | Unscheduled > 5min | L1/L2 | Resource shortage, scheduling constraints |
| **Evicted** | Pod evicted from node | L1 | Node pressure, priority preemption |
| **CPU Throttling** | Throttled > 25% of time | L1/L2 | CPU limit too low |
| **Node NotReady** | Node unavailable | L0 | Node failure, network partition |

### Namespace Priority Classification

| Priority | Response SLA | Namespaces |
|----------|-------------|------------|
| **L0** | 5-15 min | payment, order, user-auth |
| **L1** | 15-30 min | inventory, promotion, gateway |
| **L2** | 30min-2hr | monitoring, logging, batch |

---

## Phase 0: Alert Validation

### Objective
Verify alert accuracy and extract investigation context.

### Metadata Extraction

```
Cluster: ___________________
Namespace: _________________
Pod Name: __________________
Container: _________________
Workload Type: Deployment / StatefulSet / DaemonSet
Workload Name: _____________
Alert Type: ________________
Restart Count: _____________
Last State: ________________
```

### Service Priority Lookup

```sql
SELECT service_name, service_priority, owner_team, slack_channel
FROM services
WHERE k8s_namespace = '$NAMESPACE' AND k8s_deployment = '$DEPLOYMENT';
```

---

## Phase 1: Cluster & Node Health

### Objective
Verify overall cluster and node status.

### EKS MCP Tools

```
# List all nodes
list_k8s_resources(cluster_name="prod-worker01-eks-us", kind="Node", api_version="v1")

# Get specific node details
manage_k8s_resource(operation="read", cluster_name="prod-worker01-eks-us",
                    kind="Node", api_version="v1", name="$NODE_NAME")
```

### Prometheus Node Queries (Parallel)

```promql
# Node readiness
kube_node_status_condition{condition="Ready",status="true"}

# Node resource capacity
kube_node_status_allocatable{resource="memory"}
kube_node_status_allocatable{resource="cpu"}

# Node resource utilization
sum by(node)(container_memory_working_set_bytes) / sum by(node)(kube_node_status_allocatable{resource="memory"}) * 100
sum by(node)(rate(container_cpu_usage_seconds_total[5m])) / sum by(node)(kube_node_status_allocatable{resource="cpu"}) * 100
```

### Node Health Matrix

| Condition | Status | Action if Failed |
|-----------|--------|------------------|
| Ready | True | Check kubelet, network |
| MemoryPressure | False | Check memory usage |
| DiskPressure | False | Check disk space |
| PIDPressure | False | Check process count |
| NetworkUnavailable | False | Check CNI |

---

## Phase 2: Pod Investigation

### Objective
Detailed analysis of affected pod(s).

### EKS MCP Tools

```
# Get pod details
manage_k8s_resource(operation="read", cluster_name="prod-worker01-eks-us",
                    kind="Pod", api_version="v1",
                    name="$POD_NAME", namespace="$NAMESPACE")

# Get pod events
get_k8s_events(cluster_name="prod-worker01-eks-us",
               kind="Pod", name="$POD_NAME", namespace="$NAMESPACE")
```

### Prometheus Pod Queries (Parallel)

```promql
# Container status
kube_pod_container_status_restarts_total{namespace="$NS",pod="$POD"}
kube_pod_container_status_waiting_reason{namespace="$NS",pod="$POD"}
kube_pod_container_status_terminated_reason{namespace="$NS",pod="$POD"}
kube_pod_container_status_last_terminated_reason{namespace="$NS",pod="$POD"}

# Pod phase
kube_pod_status_phase{namespace="$NS",pod="$POD"}
```

### Container Status Interpretation

| Waiting Reason | Meaning | Action |
|----------------|---------|--------|
| ContainerCreating | Image pulling or volume mounting | Check image, PVC |
| CrashLoopBackOff | Repeated startup failures | Check logs, dependencies |
| ImagePullBackOff | Cannot pull image | Check registry, credentials |
| ErrImagePull | Image pull error | Check image name, tag |

| Terminated Reason | Meaning | Action |
|-------------------|---------|--------|
| OOMKilled | Memory limit exceeded | Increase limit or fix leak |
| Error | Container exited with error | Check logs for error |
| Completed | Container finished | Normal for jobs |

---

## Phase 3: Resource Analysis

### Objective
Compare actual resource usage vs limits.

### Memory Analysis (Parallel)

```promql
# Current memory usage
container_memory_working_set_bytes{namespace="$NS",pod="$POD",container!="POD"}

# Memory limit
kube_pod_container_resource_limits{namespace="$NS",pod="$POD",resource="memory"}

# Memory request
kube_pod_container_resource_requests{namespace="$NS",pod="$POD",resource="memory"}

# Memory usage trend (1h)
container_memory_working_set_bytes{namespace="$NS",pod="$POD",container!="POD"}[1h]
```

### CPU Analysis (Parallel)

```promql
# Current CPU usage
rate(container_cpu_usage_seconds_total{namespace="$NS",pod="$POD",container!="POD"}[5m])

# CPU limit
kube_pod_container_resource_limits{namespace="$NS",pod="$POD",resource="cpu"}

# CPU request
kube_pod_container_resource_requests{namespace="$NS",pod="$POD",resource="cpu"}

# CPU throttling
rate(container_cpu_cfs_throttled_seconds_total{namespace="$NS",pod="$POD"}[5m])
rate(container_cpu_cfs_periods_total{namespace="$NS",pod="$POD"}[5m])
```

### Resource Utilization Matrix

| Resource | Current | Request | Limit | Utilization | Status |
|----------|---------|---------|-------|-------------|--------|
| Memory | X Mi | Y Mi | Z Mi | X/Z % | [OK/WARN/CRIT] |
| CPU | X m | Y m | Z m | X/Z % | [OK/WARN/CRIT] |

---

## Phase 4: Workload Configuration

### Objective
Review deployment/statefulset configuration.

### EKS MCP Tools

```
# Get Deployment
manage_k8s_resource(operation="read", cluster_name="prod-worker01-eks-us",
                    kind="Deployment", api_version="apps/v1",
                    name="$DEPLOYMENT", namespace="$NAMESPACE")

# Get HPA if exists
manage_k8s_resource(operation="read", cluster_name="prod-worker01-eks-us",
                    kind="HorizontalPodAutoscaler", api_version="autoscaling/v2",
                    name="$DEPLOYMENT", namespace="$NAMESPACE")

# Get PVC if exists
list_k8s_resources(cluster_name="prod-worker01-eks-us",
                   kind="PersistentVolumeClaim", api_version="v1",
                   namespace="$NAMESPACE")
```

### Configuration Checklist

- [ ] Resource requests/limits appropriate for workload
- [ ] Replica count matches desired
- [ ] Rolling update strategy configured
- [ ] Liveness probe configured and not too aggressive
- [ ] Readiness probe configured
- [ ] PVC bound (if applicable)
- [ ] Node selector/affinity appropriate

---

## Phase 5: Application Logs

### Objective
Analyze container logs for error patterns.

### EKS MCP Tools

```
# Get current container logs
get_pod_logs(cluster_name="prod-worker01-eks-us",
             namespace="$NAMESPACE", pod_name="$POD",
             container_name="$CONTAINER", tail_lines=100)

# Get previous container logs (for crashes)
get_pod_logs(cluster_name="prod-worker01-eks-us",
             namespace="$NAMESPACE", pod_name="$POD",
             container_name="$CONTAINER", previous=true, tail_lines=100)
```

### Error Pattern Search

Look for these patterns:
- `OutOfMemoryError`, `OOM`, `killed`
- `Exception`, `Error`, `FATAL`
- `Connection refused`, `timeout`, `ECONNREFUSED`
- `Permission denied`, `access denied`
- Stack traces with line numbers

### CloudWatch Log Insights

```sql
fields @timestamp, @message
| filter kubernetes.namespace_name = "$NAMESPACE"
| filter kubernetes.pod_name like "$POD"
| filter @message like /error|exception|fatal|oom/i
| sort @timestamp desc
| limit 50
```

---

## Phase 6: Dependency Analysis

### Objective
Check health of connected services.

### Database Dependencies

```sql
-- Find database dependencies
SELECT db.host, db.database_name, db.engine
FROM service_db_mapping sdm
JOIN databases db ON sdm.db_id = db.id
WHERE sdm.service_name = '$SERVICE_NAME';
```

### Redis Dependencies

```sql
SELECT rc.cluster_name, rc.endpoint
FROM service_redis_mapping srm
JOIN redis_clusters rc ON srm.redis_id = rc.id
WHERE srm.service_name = '$SERVICE_NAME';
```

### External Service Health

```promql
# HTTP client latency to dependencies
histogram_quantile(0.99, rate(http_client_requests_seconds_bucket{app="$APP"}[5m]))

# HTTP client error rate
sum(rate(http_client_requests_seconds_count{app="$APP",status=~"5.."}[5m]))
  / sum(rate(http_client_requests_seconds_count{app="$APP"}[5m]))
```

---

## Phase 7: Alert Correlation

### Objective
Find related alerts and patterns.

### Alert History Query

```sql
SELECT alert_name, alert_value, fired_at, resolved_at, labels
FROM t_umb_alert_log
WHERE labels LIKE '%$NAMESPACE%'
  AND fired_at > NOW() - INTERVAL 24 HOUR
ORDER BY fired_at DESC
LIMIT 50;
```

### Pattern Detection

Look for:
- Multiple pods in same deployment affected
- Alerts across multiple namespaces (systemic issue)
- Recurring pattern at same time daily (scheduled job impact)
- Correlation with deployment events

---

## Phase 8: Alert-Specific Playbooks

### OOMKilled Playbook

1. **Check memory usage trend before OOM**
   ```promql
   container_memory_working_set_bytes{namespace="$NS",pod="$POD"}[1h]
   ```

2. **Compare with memory limit**
   - If usage gradually approaches limit → Memory leak
   - If sudden spike → Large request or data load

3. **Review heap dumps (for Java)**
   - Check if heap dump on OOM enabled
   - Analyze for memory leak patterns

4. **Actions**:
   - Immediate: Increase memory limit
   - Long-term: Fix memory leak or optimize memory usage

### CrashLoopBackOff Playbook

1. **Get previous container logs**
   ```
   get_pod_logs(..., previous=true)
   ```

2. **Check startup dependencies**
   - Database connectivity
   - Config maps and secrets exist
   - Required services available

3. **Verify image**
   - Correct image tag
   - Image pull successful

4. **Actions**:
   - Fix startup error based on logs
   - Verify all dependencies available
   - Check for recent deployment changes

### Pending Pod Playbook

1. **Check pod events for reason**
   ```
   get_k8s_events(...)
   ```

2. **Common reasons and fixes**:
   | Reason | Fix |
   |--------|-----|
   | Insufficient cpu | Add nodes or reduce requests |
   | Insufficient memory | Add nodes or reduce requests |
   | FailedScheduling | Check node selectors, taints |
   | PVC not bound | Check storage class, PV |

3. **Actions**:
   - Scale cluster or adjust resource requests
   - Review scheduling constraints

### CPU Throttling Playbook

1. **Check throttling metrics**
   ```promql
   rate(container_cpu_cfs_throttled_periods_total{namespace="$NS",pod="$POD"}[5m])
   / rate(container_cpu_cfs_periods_total{namespace="$NS",pod="$POD"}[5m])
   ```

2. **If throttling > 25%**:
   - CPU limit too restrictive
   - Consider increasing or removing limit

3. **Actions**:
   - Increase CPU limit
   - Optimize CPU-intensive code

---

## Phase 9: Report Generation

### Standard Report Template

```markdown
# Kubernetes Alert Investigation Report

**Report ID**: [Auto-generated]
**Timestamp**: [Investigation time]
**Investigator**: [Name/System]

## 1. Alert Summary

| Field | Value |
|-------|-------|
| Cluster | prod-worker01-eks-us |
| Namespace | [namespace] |
| Pod | [pod-name] |
| Workload | [deployment/statefulset] |
| Alert Type | [OOMKilled/CrashLoopBackOff/etc] |
| Priority | [L0/L1/L2] |
| Restart Count | [N] |

## 2. Cluster Health

| Node | Status | CPU | Memory | Pods |
|------|--------|-----|--------|------|
| [node1] | Ready | X% | Y% | N |
| [node2] | Ready | X% | Y% | N |

## 3. Pod Status

| Container | Status | Restarts | Last State | Reason |
|-----------|--------|----------|------------|--------|
| [name] | [status] | [N] | [state] | [reason] |

## 4. Resource Analysis

| Resource | Current | Request | Limit | Utilization |
|----------|---------|---------|-------|-------------|
| Memory | X Mi | Y Mi | Z Mi | X/Z % |
| CPU | X m | Y m | Z m | X/Z % |

## 5. Event Timeline

| Time | Type | Reason | Message |
|------|------|--------|---------|
| [time] | [type] | [reason] | [message] |

## 6. Log Analysis

**Error Patterns Found**:
```
[Key error messages]
```

## 7. Dependency Status

| Dependency | Type | Status | Latency |
|------------|------|--------|---------|
| [name] | [DB/Cache/API] | [OK/WARN] | [Xms] |

## 8. Root Cause Analysis

**Primary Cause**: [Determined cause]

**Evidence**:
1. [Evidence with metric/log reference]
2. [Evidence]

## 9. Recommendations

| Priority | Action | Owner | Timeline |
|----------|--------|-------|----------|
| **Immediate** | [Action] | [Team] | Now |
| **Short-term** | [Action] | [Team] | This week |
| **Long-term** | [Action] | [Team] | This quarter |
```

---

## Appendix: Data Source Reference

### EKS Cluster

| Cluster | Region | Purpose |
|---------|--------|---------|
| prod-worker01-eks-us | us-east-1 | Production workloads |

### Prometheus Datasource

| Name | UID | Purpose |
|------|-----|---------|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | kube-state-metrics, cAdvisor |

### Key Namespaces

| Namespace | Priority | Owner |
|-----------|----------|-------|
| payment | L0 | payment-team |
| order | L0 | order-team |
| inventory | L1 | supply-team |
| monitoring | L2 | devops-team |

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.0 | 2025-01-19 | DevOps | EKS MCP integration, playbooks |
| 1.0 | 2025-01-11 | DevOps | Initial release |
