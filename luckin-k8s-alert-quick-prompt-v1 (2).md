# K8s Alert Investigation - Quick Prompt v1.0
## Copy → Fill Placeholders → Run in Claude Code

---

```markdown
# Kubernetes Alert Investigation - Pod/Cluster Diagnosis

## Alert Context
- **Alert Name:** [PASTE_ALERT_NAME]
- **Metric:** [PASTE_METRIC]
- **Time:** [YYYY-MM-DD HH:MM TZ] (UTC: [XX:XX])
- **Cluster:** [EKS cluster name]
- **Namespace:** [namespace]
- **Workload:** [Deployment/StatefulSet name]
- **Pod(s):** [pod name(s)]
- **Status:** [Active/Resolved]

---

## Phase 0: Alert Type Identification

| Alert Pattern | Focus |
|---------------|-------|
| OOMKilled | Memory limits, leak |
| CrashLoopBackOff | App error, startup issue |
| Pending | Resources, scheduling |
| Evicted | Node pressure |

Determine service level:
- L1: dify-*, milvus-*, meta-apiserver
- L2: hello-world, monitoring-*, test

---

## Phase 1: Node Health

**Prometheus** (UMBQuerier-Luckin):
```promql
# Node ready status
kube_node_status_condition{condition="Ready", status="true"}

# Node pressure
kube_node_status_condition{condition="MemoryPressure", status="true"}
kube_node_status_condition{condition="DiskPressure", status="true"}
```

**kubectl** (via EKS MCP):
```bash
kubectl get nodes
kubectl top nodes
```

---

## Phase 2: Pod Investigation

### 2.1 Pod Status
```bash
kubectl get pod [POD] -n [NAMESPACE] -o wide
kubectl describe pod [POD] -n [NAMESPACE]
```

**Prometheus:**
```promql
kube_pod_container_status_restarts_total{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}
kube_pod_container_status_last_terminated_reason{namespace="[NAMESPACE]"}
```

### 2.2 OOMKilled Check 🔴 MOST COMMON
```bash
kubectl get pod [POD] -n [NAMESPACE] -o jsonpath='{.status.containerStatuses[*].lastState}'
kubectl describe pod [POD] -n [NAMESPACE] | grep -A5 "Last State"
kubectl get events -n [NAMESPACE] --field-selector involvedObject.name=[POD] | grep -i oom
```

**Prometheus:**
```promql
kube_pod_container_status_last_terminated_reason{reason="OOMKilled", namespace="[NAMESPACE]"}
```

### 2.3 Logs
```bash
kubectl logs [POD] -n [NAMESPACE] -c [CONTAINER]
kubectl logs [POD] -n [NAMESPACE] -c [CONTAINER] --previous
kubectl logs [POD] -n [NAMESPACE] --tail=100 | grep -i "error\|exception\|oom"
```

---

## Phase 3: Resource Analysis

**Prometheus:**
```promql
# Memory usage vs limit
container_memory_working_set_bytes{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}
/ kube_pod_container_resource_limits{resource="memory", namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"} * 100

# CPU usage vs limit
rate(container_cpu_usage_seconds_total{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}[5m])
/ kube_pod_container_resource_limits{resource="cpu", namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"} * 100

# Approaching limit (>80%)
container_memory_working_set_bytes{container!="POD"} / kube_pod_container_resource_limits{resource="memory"} * 100 > 80
```

```bash
kubectl top pod [POD] -n [NAMESPACE]
kubectl top pods -n [NAMESPACE] --sort-by=memory
```

---

## Phase 4: Workload Check

```bash
# Deployment/StatefulSet
kubectl get deployment [DEPLOY] -n [NAMESPACE]
kubectl describe deployment [DEPLOY] -n [NAMESPACE]
kubectl rollout history deployment/[DEPLOY] -n [NAMESPACE]

# HPA
kubectl get hpa -n [NAMESPACE]

# PVC
kubectl get pvc -n [NAMESPACE]
```

**Prometheus:**
```promql
kube_deployment_status_replicas_available{namespace="[NAMESPACE]", deployment="[DEPLOY]"}
kube_deployment_status_replicas_unavailable{namespace="[NAMESPACE]", deployment="[DEPLOY]"}
```

---

## Phase 5: Dependencies

### Dify Workloads → PostgreSQL
| Workload | Database |
|----------|----------|
| dify-api | aws-luckyus-dify-rw |
| dify-plugin-daemon | aws-luckyus-dify-rw |

**PostgreSQL Check:**
```sql
SELECT COUNT(*) FROM pg_stat_activity;
SELECT usename, application_name, COUNT(*) FROM pg_stat_activity GROUP BY 1,2;
```

---

## Phase 6: Alert History

**MySQL** (aws-luckyus-devops-rw):
```sql
SELECT alert_name, instance, create_time, resolve_time
FROM t_umb_alert_log
WHERE alert_name LIKE '%K8s%' OR alert_name LIKE '%pod%' OR instance LIKE '%[NAMESPACE]%'
ORDER BY create_time DESC LIMIT 20;
```

---

## Output Format

### Summary
| Field | Value |
|-------|-------|
| Root Cause | |
| Cluster | |
| Namespace | |
| Workload | |
| Pod(s) | |
| Alert Type | OOMKilled / CrashLoop / etc. |
| Service Level | L1/L2 |
| Resolution | |

### Resource Analysis
| Resource | Current | Request | Limit | Status |
|----------|---------|---------|-------|--------|
| Memory | MB | MB | MB | ✅/⚠️/🔴 |
| CPU | m | m | m | ✅/⚠️/🔴 |

### Pod Status
| Pod | Status | Restarts | Node |
|-----|--------|----------|------|
| | | | |

### Key Logs
```
[Error messages]
```

### Recommendations
```

---

## Quick Reference

### Historical K8s Incidents
| Workload | Issue | Duration | Resolution |
|----------|-------|----------|------------|
| dify-plugin-daemon | OOM | 3 min | Auto-restart |
| hello-world | Stuck | 5+ hours | Manual delete |
| milvus (5 pods) | Memory | 17+ hours | Manual scale |
| dify-api | OOM | 1 min | Auto-restart |

### Emergency Commands
```bash
# Delete stuck pod
kubectl delete pod [POD] -n [NAMESPACE] --force --grace-period=0

# Rollback
kubectl rollout undo deployment/[DEPLOY] -n [NAMESPACE]

# Quick scale down/up
kubectl scale deployment/[DEPLOY] -n [NAMESPACE] --replicas=0
kubectl scale deployment/[DEPLOY] -n [NAMESPACE] --replicas=3

# Patch memory limit
kubectl patch deployment [DEPLOY] -n [NAMESPACE] -p '{"spec":{"template":{"spec":{"containers":[{"name":"[CONTAINER]","resources":{"limits":{"memory":"2Gi"}}}]}}}}'
```

### Common OOM Root Causes
1. Memory limit too low for workload
2. Memory leak in application
3. Large dataset processing
4. Traffic spike

---

## Checklist

- [ ] Alert type identified (OOMKilled, CrashLoop, etc.)
- [ ] Node health verified
- [ ] Pod status and lastState checked
- [ ] Container logs reviewed (current + previous)
- [ ] Resource usage vs limits analyzed
- [ ] Workload config checked (replicas, HPA)
- [ ] Dependencies verified (DB, Redis)
- [ ] Alert history correlated
- [ ] Root cause identified
- [ ] Owner notified
- [ ] Recommendations documented
