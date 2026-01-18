# Luckin Coffee DevOps Alert Investigation SOP
## Kubernetes (EKS) Diagnosis Edition v1.0
### Integrated with MCP Data Sources & System Architecture

---

## Document Info

| Field | Value |
|-------|-------|
| Version | 1.0 |
| Last Updated | 2025-01-11 |
| Scope | Kubernetes/EKS Alerts (Pod, Node, Cluster) |
| Related SOPs | EC2 SOP v4.0, RDS SOP v1.0 |

---

## Available MCP Data Sources for K8s Investigation

### AWS EKS (via MCP)

| Capability | Description |
|------------|-------------|
| Cluster Management | List clusters, describe cluster status |
| Kubernetes Resources | Pods, Deployments, Services, ConfigMaps, Secrets |
| Pod Logs | Container logs, previous container logs |
| Events | Kubernetes events (warnings, errors) |
| Metrics | Resource usage, HPA status |
| Nodes | Node status, capacity, allocatable |

### Prometheus Datasources

| Datasource | UID | Use For |
|------------|-----|---------|
| **UMBQuerier-Luckin** | df8o21agxtkw0d | K8s metrics, pod metrics, node metrics |
| prometheus | ff7hkeec6c9a8e | General metrics |

### AWS CloudWatch (via MCP)

| Service | K8s Investigation Use |
|---------|----------------------|
| CloudWatch Logs | Container logs, cluster logs |
| CloudWatch Metrics | EKS cluster metrics, Container Insights |
| CloudWatch Alarms | K8s-related alarms |

### Grafana Ecosystem

| Feature | K8s Use |
|---------|---------|
| Dashboards | K8s monitoring dashboards |
| Alerting | Pod/node alert rules |
| Sift | Error pattern analysis |

---

## Historical K8s Incidents (From Alert Patterns)

| Date | Service | Duration | Resolution | Root Cause Pattern |
|------|---------|----------|------------|-------------------|
| Sep 30, 2025 | dify-plugin-daemon | 3 min | Auto-resolved | Pod OOM, auto-restart |
| Sep 23, 2025 | hello-world pod | 5+ hours | Manual intervention | Stuck pod, required delete |
| Jul 15, 2025 | meta-apiserver | 4+ hours | Manual intervention | Resource contention |
| Jul 2, 2025 | dify-api | 1 min | Auto-resolved | Brief OOM spike |
| May 20, 2025 | milvus cluster (5 pods) | 17+ hours | Manual intervention | Memory exhaustion, required scaling |

**Key Pattern:** Most P1 K8s alerts are **pod-level OOM events**, not node-level issues.

---

## Quick Start Checklist

Before investigation:
- [ ] Identify cluster name (EKS cluster)
- [ ] Identify namespace
- [ ] Identify workload type (Deployment, StatefulSet, DaemonSet)
- [ ] Identify pod name(s) affected
- [ ] Check if this is OOM, CrashLoopBackOff, or other issue
- [ ] Determine service level based on workload
- [ ] Identify service owner

---

## Service Level Priority Matrix

| Level | Definition | Response Time | Example Workloads |
|-------|------------|---------------|-------------------|
| **L0** 🔴 | Core Business | <15 min | Production APIs, payment services |
| **L1** 🟡 | Important | <30 min | Dify platform, internal tools |
| **L2** 🟢 | Normal | <2 hours | Dev/test workloads, monitoring |

### Known K8s Workloads by Priority

| Priority | Workloads |
|----------|-----------|
| L0 | Production-critical pods (if any) |
| L1 | dify-api, dify-plugin-daemon, milvus, meta-apiserver |
| L2 | hello-world, test deployments, monitoring pods |

---

## Copy-Paste Investigation Prompt

```markdown
# Kubernetes Alert Investigation - Pod/Cluster Diagnosis

## Alert Context
- **Alert Name:** [PASTE_FULL_ALERT_NAME - e.g., 【K8s】P1 Pod OOMKilled]
- **Alert Metric:** [PASTE_METRIC]
- **Trigger Time:** [YYYY-MM-DD HH:MM TIMEZONE] (UTC: [XX:XX])
- **Duration:** [Duration or "ongoing"]
- **Status:** [Active / Resolved / Flapping]
- **Environment:** Luckin-PROD
- **Cluster:** [EKS cluster name]
- **Namespace:** [namespace]
- **Workload:** [Deployment/StatefulSet/DaemonSet name]
- **Pod(s):** [pod name(s)]

---

## PHASE 0: Alert Validation & Pod Identification

### 0.1 Identify Affected Resources

**From Alert, determine:**
- Cluster: [EKS cluster name]
- Namespace: [namespace]
- Workload Type: [Deployment / StatefulSet / DaemonSet / Job]
- Workload Name: [name]
- Pod Name(s): [specific pod(s)]
- Container(s): [if multi-container pod]

### 0.2 Determine Alert Type

| Alert Pattern | Investigation Focus |
|---------------|---------------------|
| OOMKilled | Memory limits, memory leak |
| CrashLoopBackOff | Application error, startup issue |
| Pending | Resource constraints, scheduling |
| ImagePullBackOff | Image registry, credentials |
| Evicted | Node resource pressure |
| NodeNotReady | Node health issue |

### 0.3 Determine Service Level

| Workload | Service Level | Response |
|----------|---------------|----------|
| dify-api | L1 🟡 | <30 min |
| dify-plugin-daemon | L1 🟡 | <30 min |
| milvus-* | L1 🟡 | <30 min |
| meta-apiserver | L1 🟡 | <30 min |
| hello-world | L2 🟢 | <2 hours |
| monitoring-* | L2 🟢 | <2 hours |

---

## PHASE 1: Cluster & Node Health Check

### 1.1 Cluster Status

**AWS EKS MCP:**
- Describe cluster status
- Check cluster version
- Verify cluster endpoint accessibility

### 1.2 Node Health

**Prometheus** (UMBQuerier-Luckin):
```promql
# Node status (1 = Ready)
kube_node_status_condition{condition="Ready", status="true"}

# Node resource pressure
kube_node_status_condition{condition="MemoryPressure", status="true"}
kube_node_status_condition{condition="DiskPressure", status="true"}
kube_node_status_condition{condition="PIDPressure", status="true"}

# Node capacity
kube_node_status_capacity{resource="cpu"}
kube_node_status_capacity{resource="memory"}
kube_node_status_capacity{resource="pods"}

# Node allocatable
kube_node_status_allocatable{resource="cpu"}
kube_node_status_allocatable{resource="memory"}

# Node CPU usage
sum by(node) (rate(node_cpu_seconds_total{mode!="idle"}[5m])) 
/ 
sum by(node) (kube_node_status_allocatable{resource="cpu"})

# Node memory usage
sum by(node) (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
/
sum by(node) (kube_node_status_allocatable{resource="memory"}) * 100
```

**kubectl commands** (via EKS MCP):
```bash
# Node status
kubectl get nodes
kubectl describe node [node-name]

# Node resource usage
kubectl top nodes

# Events on node
kubectl get events --field-selector involvedObject.kind=Node
```

### 1.3 Identify Node Running Affected Pod

```bash
# Find which node the pod is/was on
kubectl get pod [pod-name] -n [namespace] -o wide

# If pod was terminated, check events
kubectl get events -n [namespace] --field-selector involvedObject.name=[pod-name]
```

---

## PHASE 2: Pod-Level Investigation

### 2.1 Pod Status

**kubectl commands** (via EKS MCP):
```bash
# Pod status
kubectl get pod [pod-name] -n [namespace] -o wide

# Detailed pod description
kubectl describe pod [pod-name] -n [namespace]

# Pod YAML
kubectl get pod [pod-name] -n [namespace] -o yaml
```

**Prometheus** (UMBQuerier-Luckin):
```promql
# Pod phase
kube_pod_status_phase{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}

# Pod ready
kube_pod_status_ready{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}

# Container restarts
kube_pod_container_status_restarts_total{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}

# Restart trend
increase(kube_pod_container_status_restarts_total{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}[1h])

# Container status
kube_pod_container_status_running{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}
kube_pod_container_status_waiting{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}
kube_pod_container_status_terminated{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}
```

### 2.2 OOMKilled Analysis 🔴 MOST COMMON K8S ISSUE

**Check if OOMKilled:**
```bash
# Check container last state
kubectl get pod [pod-name] -n [namespace] -o jsonpath='{.status.containerStatuses[*].lastState}'

# Detailed termination reason
kubectl describe pod [pod-name] -n [namespace] | grep -A5 "Last State"

# Look for OOMKilled in events
kubectl get events -n [namespace] --field-selector involvedObject.name=[pod-name] | grep -i oom
```

**Prometheus** (UMBQuerier-Luckin):
```promql
# OOMKilled containers
kube_pod_container_status_last_terminated_reason{reason="OOMKilled", namespace="[NAMESPACE]"}

# All OOMKilled in cluster (recent)
increase(kube_pod_container_status_restarts_total{namespace="[NAMESPACE]"}[1h]) 
and on(pod, container) 
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
```

**Root Cause Analysis for OOM:**
1. Memory limit too low?
2. Memory leak in application?
3. Unexpected traffic spike?
4. Large dataset processing?

### 2.3 CrashLoopBackOff Analysis

**Check container logs:**
```bash
# Current container logs
kubectl logs [pod-name] -n [namespace] -c [container-name]

# Previous container logs (if restarted)
kubectl logs [pod-name] -n [namespace] -c [container-name] --previous

# Tail recent logs
kubectl logs [pod-name] -n [namespace] -c [container-name] --tail=100

# Follow logs (for ongoing issue)
kubectl logs [pod-name] -n [namespace] -c [container-name] -f
```

**Prometheus**:
```promql
# Containers in CrashLoopBackOff
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff", namespace="[NAMESPACE]"}

# High restart count
kube_pod_container_status_restarts_total{namespace="[NAMESPACE]"} > 5
```

**Common CrashLoopBackOff causes:**
- Application startup error
- Missing configuration/secrets
- Database connection failure
- Dependency not available
- Permission issues

### 2.4 Pending Pod Analysis

**Check why pod is pending:**
```bash
kubectl describe pod [pod-name] -n [namespace] | grep -A10 "Events"

# Check conditions
kubectl get pod [pod-name] -n [namespace] -o jsonpath='{.status.conditions}'
```

**Prometheus**:
```promql
# Pending pods
kube_pod_status_phase{phase="Pending", namespace="[NAMESPACE]"}

# Unschedulable pods
kube_pod_status_unschedulable{namespace="[NAMESPACE]"}
```

**Common Pending causes:**
- Insufficient CPU/memory on nodes
- Node selector/affinity mismatch
- PVC not bound
- Taints/tolerations issue

---

## PHASE 3: Resource Analysis

### 3.1 Container Resource Usage vs Limits

**Prometheus** (UMBQuerier-Luckin):
```promql
# Container memory usage
container_memory_working_set_bytes{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}

# Container memory limit
kube_pod_container_resource_limits{resource="memory", namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}

# Memory usage percentage (vs limit)
container_memory_working_set_bytes{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}
/
kube_pod_container_resource_limits{resource="memory", namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"} * 100

# Container CPU usage
rate(container_cpu_usage_seconds_total{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}[5m])

# CPU limit
kube_pod_container_resource_limits{resource="cpu", namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"}

# CPU usage percentage (vs limit)
rate(container_cpu_usage_seconds_total{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}[5m])
/
kube_pod_container_resource_limits{resource="cpu", namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*"} * 100

# Containers approaching memory limit (>80%)
container_memory_working_set_bytes{container!="POD"}
/ 
kube_pod_container_resource_limits{resource="memory"} * 100 > 80
```

**kubectl**:
```bash
# Pod resource usage
kubectl top pod [pod-name] -n [namespace]

# All pods in namespace
kubectl top pods -n [namespace] --sort-by=memory
kubectl top pods -n [namespace] --sort-by=cpu
```

### 3.2 Resource Requests vs Allocatable

**Prometheus**:
```promql
# Total memory requests in namespace
sum(kube_pod_container_resource_requests{resource="memory", namespace="[NAMESPACE]"})

# Total memory limits in namespace
sum(kube_pod_container_resource_limits{resource="memory", namespace="[NAMESPACE]"})

# Namespace resource quota usage
kube_resourcequota{namespace="[NAMESPACE]", type="used"}
kube_resourcequota{namespace="[NAMESPACE]", type="hard"}
```

### 3.3 Historical Resource Trends

**Prometheus**:
```promql
# Memory usage over last 24 hours
container_memory_working_set_bytes{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}[24h]

# Max memory usage
max_over_time(container_memory_working_set_bytes{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}[24h])

# Memory growth rate (potential leak detection)
deriv(container_memory_working_set_bytes{namespace="[NAMESPACE]", pod=~"[POD_PREFIX].*", container!="POD"}[1h])
```

---

## PHASE 4: Workload Configuration Check

### 4.1 Deployment/StatefulSet Status

**kubectl**:
```bash
# Deployment status
kubectl get deployment [deployment-name] -n [namespace]
kubectl describe deployment [deployment-name] -n [namespace]

# StatefulSet status
kubectl get statefulset [statefulset-name] -n [namespace]
kubectl describe statefulset [statefulset-name] -n [namespace]

# Replica status
kubectl get deployment [deployment-name] -n [namespace] -o jsonpath='{.status}'

# Rollout status
kubectl rollout status deployment/[deployment-name] -n [namespace]

# Rollout history
kubectl rollout history deployment/[deployment-name] -n [namespace]
```

**Prometheus**:
```promql
# Deployment replicas
kube_deployment_status_replicas{namespace="[NAMESPACE]", deployment="[DEPLOYMENT]"}
kube_deployment_status_replicas_available{namespace="[NAMESPACE]", deployment="[DEPLOYMENT]"}
kube_deployment_status_replicas_unavailable{namespace="[NAMESPACE]", deployment="[DEPLOYMENT]"}

# StatefulSet replicas
kube_statefulset_status_replicas{namespace="[NAMESPACE]", statefulset="[STATEFULSET]"}
kube_statefulset_status_replicas_ready{namespace="[NAMESPACE]", statefulset="[STATEFULSET]"}
```

### 4.2 HPA (Horizontal Pod Autoscaler) Check

**kubectl**:
```bash
# HPA status
kubectl get hpa -n [namespace]
kubectl describe hpa [hpa-name] -n [namespace]
```

**Prometheus**:
```promql
# HPA current replicas
kube_horizontalpodautoscaler_status_current_replicas{namespace="[NAMESPACE]"}

# HPA desired replicas
kube_horizontalpodautoscaler_status_desired_replicas{namespace="[NAMESPACE]"}

# HPA min/max
kube_horizontalpodautoscaler_spec_min_replicas{namespace="[NAMESPACE]"}
kube_horizontalpodautoscaler_spec_max_replicas{namespace="[NAMESPACE]"}

# HPA at max (can't scale more)
kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_max_replicas
```

### 4.3 PVC (Persistent Volume Claim) Check

**kubectl**:
```bash
# PVC status
kubectl get pvc -n [namespace]
kubectl describe pvc [pvc-name] -n [namespace]

# PV status
kubectl get pv | grep [namespace]
```

**Prometheus**:
```promql
# PVC usage (if exposed)
kubelet_volume_stats_used_bytes{namespace="[NAMESPACE]"}
kubelet_volume_stats_capacity_bytes{namespace="[NAMESPACE]"}

# PVC usage percentage
kubelet_volume_stats_used_bytes{namespace="[NAMESPACE]"}
/
kubelet_volume_stats_capacity_bytes{namespace="[NAMESPACE]"} * 100
```

---

## PHASE 5: Application Logs & Events

### 5.1 Container Logs

**kubectl**:
```bash
# Current logs
kubectl logs [pod-name] -n [namespace] -c [container-name]

# Previous container (after restart)
kubectl logs [pod-name] -n [namespace] -c [container-name] --previous

# All containers in pod
kubectl logs [pod-name] -n [namespace] --all-containers=true

# Logs with timestamps
kubectl logs [pod-name] -n [namespace] --timestamps=true

# Last 100 lines
kubectl logs [pod-name] -n [namespace] --tail=100

# Logs since specific time
kubectl logs [pod-name] -n [namespace] --since=1h
kubectl logs [pod-name] -n [namespace] --since-time="2025-01-10T17:00:00Z"
```

**AWS CloudWatch Logs** (if Container Insights enabled):
```
Log Group: /aws/containerinsights/[cluster-name]/application
Log Group: /aws/containerinsights/[cluster-name]/host

# Log Insights query
fields @timestamp, @message, kubernetes.pod_name
| filter kubernetes.namespace_name = "[NAMESPACE]"
| filter kubernetes.pod_name like /[POD_PREFIX]/
| filter @message like /ERROR|Exception|OOM/
| sort @timestamp desc
| limit 100
```

### 5.2 Kubernetes Events

**kubectl**:
```bash
# Events for specific pod
kubectl get events -n [namespace] --field-selector involvedObject.name=[pod-name]

# All events in namespace (sorted by time)
kubectl get events -n [namespace] --sort-by='.lastTimestamp'

# Warning events only
kubectl get events -n [namespace] --field-selector type=Warning

# Events in last hour
kubectl get events -n [namespace] --field-selector 'lastTimestamp>2025-01-10T17:00:00Z'
```

**Prometheus**:
```promql
# Kubernetes events (if exported)
kube_event_count{namespace="[NAMESPACE]", reason=~"Failed|OOMKilled|Evicted|BackOff"}
```

### 5.3 Error Pattern Analysis

**Search for common error patterns in logs:**
```bash
# OOM related
kubectl logs [pod-name] -n [namespace] --previous | grep -i "out of memory\|oom\|killed"

# Connection errors
kubectl logs [pod-name] -n [namespace] | grep -i "connection refused\|timeout\|ECONNREFUSED"

# Startup errors
kubectl logs [pod-name] -n [namespace] | grep -i "error\|fatal\|panic" | head -50

# Database connection issues
kubectl logs [pod-name] -n [namespace] | grep -i "mysql\|postgres\|redis\|connection"
```

---

## PHASE 6: Dependencies Check

### 6.1 Database Dependency

**If pod connects to RDS, check database health:**

| K8s Workload | Database Dependency |
|--------------|---------------------|
| dify-api | aws-luckyus-dify-rw (PostgreSQL) |
| dify-plugin-daemon | aws-luckyus-dify-rw (PostgreSQL) |
| Other services | See RDS SOP mapping |

**PostgreSQL Check** (aws-luckyus-dify-rw):
```sql
-- Connection status
SELECT COUNT(*) FROM pg_stat_activity;

-- Active queries
SELECT pid, usename, state, query_start, LEFT(query, 100)
FROM pg_stat_activity
WHERE state = 'active';
```

### 6.2 Redis/Cache Dependency

**If pod uses Redis cache:**

**Redis Commands**:
```
INFO memory
INFO clients
PING
```

**Prometheus** (prometheus_redis):
```promql
redis_up{instance=~".*[REDIS_CLUSTER].*"}
redis_connected_clients{instance=~".*[REDIS_CLUSTER].*"}
```

### 6.3 Other Service Dependencies

**Check if dependent services are healthy:**
```bash
# Check other pods in same namespace
kubectl get pods -n [namespace]

# Check services
kubectl get svc -n [namespace]

# Test connectivity from within cluster (if needed)
kubectl exec -it [pod-name] -n [namespace] -- curl [service-name]:[port]/health
```

---

## PHASE 7: Alert Correlation

### 7.1 Alert History

**MySQL** (aws-luckyus-devops-rw):
```sql
-- K8s alerts for this workload
SELECT 
    alert_name,
    instance,
    alert_status,
    create_time,
    resolve_time,
    TIMESTAMPDIFF(MINUTE, create_time, COALESCE(resolve_time, NOW())) as duration_min
FROM t_umb_alert_log
WHERE alert_name LIKE '%K8s%'
   OR alert_name LIKE '%pod%'
   OR alert_name LIKE '%container%'
   OR instance LIKE '%[POD_PREFIX]%'
   OR instance LIKE '%[NAMESPACE]%'
ORDER BY create_time DESC
LIMIT 30;

-- Correlated alerts in same timeframe
SELECT 
    alert_name,
    instance,
    create_time
FROM t_umb_alert_log
WHERE create_time BETWEEN DATE_SUB('[INCIDENT_TIME]', INTERVAL 30 MINUTE) 
                      AND DATE_ADD('[INCIDENT_TIME]', INTERVAL 30 MINUTE)
ORDER BY create_time;

-- Recurring pattern for this workload
SELECT 
    DATE(create_time) as date,
    alert_name,
    COUNT(*) as occurrences
FROM t_umb_alert_log
WHERE (instance LIKE '%[POD_PREFIX]%' OR alert_name LIKE '%[WORKLOAD]%')
  AND create_time > DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY DATE(create_time), alert_name
ORDER BY date DESC;
```

### 7.2 Node-Level Correlation

**Check if node had issues around same time:**

**Prometheus**:
```promql
# Node memory pressure during incident
kube_node_status_condition{condition="MemoryPressure", status="true"}

# Node disk pressure
kube_node_status_condition{condition="DiskPressure", status="true"}

# Node not ready events
changes(kube_node_status_condition{condition="Ready", status="true"}[1h])
```

---

## PHASE 8: Workload-Specific Playbooks

### 8.1 Dify Platform (dify-api, dify-plugin-daemon)

**Common Issues:**
- Memory pressure from AI model loading
- PostgreSQL connection issues
- Plugin initialization failures

**Specific Checks:**
```bash
# Dify logs
kubectl logs -l app=dify-api -n [namespace] --tail=100
kubectl logs -l app=dify-plugin-daemon -n [namespace] --tail=100

# PostgreSQL connectivity
kubectl exec -it [dify-pod] -n [namespace] -- psql -h [pg-host] -U [user] -c "SELECT 1"
```

**Database Check** (aws-luckyus-dify-rw or aws-luckyus-difynew-rw):
```sql
-- Active connections from Dify
SELECT usename, application_name, COUNT(*)
FROM pg_stat_activity
WHERE application_name LIKE '%dify%'
GROUP BY usename, application_name;
```

### 8.2 Milvus Cluster

**Common Issues:**
- Memory exhaustion from vector operations
- Disk space for index storage
- Multi-pod coordination issues

**Specific Checks:**
```bash
# All Milvus pods
kubectl get pods -n [namespace] -l app=milvus

# Milvus component status
kubectl logs -l app=milvus -l component=proxy -n [namespace] --tail=100
kubectl logs -l app=milvus -l component=datanode -n [namespace] --tail=100
```

### 8.3 Meta-Apiserver

**Common Issues:**
- Resource contention with other pods
- etcd connectivity
- Certificate issues

**Specific Checks:**
```bash
# API server logs
kubectl logs -l component=apiserver -n [namespace] --tail=100

# Certificate expiry (if applicable)
kubectl get certificate -n [namespace]
```

---

## PHASE 9: Output Report Format

### Executive Summary
| Field | Value |
|-------|-------|
| **Root Cause** | [One-line summary] |
| **Cluster** | [EKS cluster name] |
| **Namespace** | [namespace] |
| **Workload** | [Deployment/StatefulSet name] |
| **Pod(s)** | [pod name(s)] |
| **Service Level** | [L0/L1/L2] |
| **Owner** | [Name] |
| **Impact** | [What was affected] |
| **Resolution** | [How resolved] |
| **Status** | ✅ Resolved / ⚠️ Monitoring / 🔴 Active |

### Alert Type Identification
| Item | Found |
|------|-------|
| Alert Type | [OOMKilled / CrashLoopBackOff / Pending / etc.] |
| Container | [container name] |
| Termination Reason | [from lastState] |
| Exit Code | [if available] |

### Resource Analysis
| Resource | Current | Request | Limit | Status |
|----------|---------|---------|-------|--------|
| Memory | [X] MB | [X] MB | [X] MB | ✅/⚠️/🔴 |
| CPU | [X] m | [X] m | [X] m | ✅/⚠️/🔴 |

### Pod Status
| Pod | Status | Restarts | Node | Age |
|-----|--------|----------|------|-----|
| [pod-1] | [status] | [count] | [node] | [age] |

### Node Health (if relevant)
| Node | Status | Memory Pressure | Disk Pressure | CPU % |
|------|--------|-----------------|---------------|-------|
| [node] | [Ready/NotReady] | [True/False] | [True/False] | [X]% |

### Dependencies Status
| Dependency | Type | Status | Notes |
|------------|------|--------|-------|
| [db-name] | PostgreSQL | ✅/⚠️/🔴 | |
| [redis] | Redis | ✅/⚠️/🔴 | |
| [service] | K8s Service | ✅/⚠️/🔴 | |

### Key Log Findings
```
[Relevant error messages from logs]
```

### Root Cause Analysis
[Detailed explanation]

### Resolution Steps
1. [Step taken]
2. [Step taken]

### Preventive Recommendations

**Priority 1 - Immediate**
| Action | Owner | Impact |
|--------|-------|--------|
| Increase memory limit | DevOps | Prevent OOM |
| Add resource requests | DevOps | Better scheduling |

**Priority 2 - Short Term**
| Action | Owner | Impact |
|--------|-------|--------|

### Monitoring Improvements
- [ ] [New alert rule for memory approaching limit]
- [ ] [Dashboard for workload resources]

---

## Appendix A: Common K8s Issues & Quick Fixes

| Issue | Quick Fix | Long-term Solution |
|-------|-----------|-------------------|
| **OOMKilled** | Increase memory limit | Profile app memory, fix leak |
| **CrashLoopBackOff** | Check logs `--previous` | Fix application error |
| **Pending (resources)** | Scale down other pods | Add nodes, adjust requests |
| **Pending (PVC)** | Check PV availability | Review storage class |
| **ImagePullBackOff** | Check image name/tag | Fix registry credentials |
| **Evicted** | Clear node disk space | Add node monitoring |

---

## Appendix B: Emergency Commands

```bash
# Restart pod (delete and let controller recreate)
kubectl delete pod [pod-name] -n [namespace]

# Force delete stuck pod
kubectl delete pod [pod-name] -n [namespace] --force --grace-period=0

# Rollback deployment
kubectl rollout undo deployment/[deployment-name] -n [namespace]

# Scale deployment (quick relief)
kubectl scale deployment/[deployment-name] -n [namespace] --replicas=0
kubectl scale deployment/[deployment-name] -n [namespace] --replicas=3

# Patch resource limits (temporary increase)
kubectl patch deployment [deployment-name] -n [namespace] -p '{"spec":{"template":{"spec":{"containers":[{"name":"[container]","resources":{"limits":{"memory":"2Gi"}}}]}}}}'

# Drain node (for node issues)
kubectl drain [node-name] --ignore-daemonsets --delete-emptydir-data

# Uncordon node
kubectl uncordon [node-name]
```

---

## Appendix C: Key Prometheus Metrics for K8s

```yaml
# Pod Status
kube_pod_status_phase
kube_pod_status_ready
kube_pod_container_status_restarts_total
kube_pod_container_status_running
kube_pod_container_status_waiting
kube_pod_container_status_terminated
kube_pod_container_status_last_terminated_reason

# Resource Usage
container_memory_working_set_bytes
container_cpu_usage_seconds_total

# Resource Limits/Requests
kube_pod_container_resource_limits
kube_pod_container_resource_requests

# Node Status
kube_node_status_condition
kube_node_status_capacity
kube_node_status_allocatable

# Deployment/StatefulSet
kube_deployment_status_replicas
kube_deployment_status_replicas_available
kube_statefulset_status_replicas_ready

# HPA
kube_horizontalpodautoscaler_status_current_replicas
kube_horizontalpodautoscaler_status_desired_replicas

# PVC
kubelet_volume_stats_used_bytes
kubelet_volume_stats_capacity_bytes
```

---

## Appendix D: Workload Owner Reference

| Workload | Owner | Escalation |
|----------|-------|------------|
| dify-* | Platform Team | |
| milvus-* | BigData Team | |
| monitoring-* | SRE Team | |
| hello-world | Test (no escalation) | |

---

## Appendix E: Investigation Completion Checklist

- [ ] **Phase 0:** Alert type identified, service level determined
- [ ] **Phase 1:** Cluster and node health verified
- [ ] **Phase 2:** Pod status and termination reason identified
- [ ] **Phase 3:** Resource usage vs limits analyzed
- [ ] **Phase 4:** Workload configuration checked (Deployment, HPA, PVC)
- [ ] **Phase 5:** Container logs and events reviewed
- [ ] **Phase 6:** Dependencies (DB, Redis) checked
- [ ] **Phase 7:** Alert correlation completed
- [ ] **Phase 8:** Workload-specific checks done
- [ ] **Phase 9:** Report generated with:
  - [ ] Root cause identified
  - [ ] Resource analysis documented
  - [ ] Dependencies verified
  - [ ] Owner notified
  - [ ] Recommendations provided

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-11 | Initial K8s SOP with EKS MCP integration, OOMKilled focus, workload-specific playbooks |
```
