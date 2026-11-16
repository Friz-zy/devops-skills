---
name: kubernetes-operations
description: Plan, monitor, troubleshoot, scale, release, and recover production Kubernetes applications. Use for SLOs/alerts, logs/metrics/traces, autoscaling, rollouts, rollbacks, PodDisruptionBudgets, capacity/HA, incidents, upgrades, backups, restores, and stateful recovery.
---

# Kubernetes production operations

Operate against explicit **SLOs and recovery objectives**. Diagnose before changing production; prefer reversible, auditable actions and the existing GitOps controller. Kubernetes restarts Pods, but does not provide backups, application-level consistency or zero downtime automatically.

## Workflow

1. Identify owner, environment, SLOs, rollout mechanism, cluster/CNI/CSI/observability stack, dependencies, recent changes, current topology, resource budgets, and known runbooks.
2. Measure symptom and impact using golden signals and K8s events; separate application, DNS/network, storage, scheduling, node and control-plane failures.
3. Choose the least disruptive fix; consider rollback, traffic reduction, scale, rate limits and dependency health. For stateful changes, protect data integrity before action.
4. Validate service health, record incident timeline and corrective work; update tests, alerts, resource sizing and runbooks. Do not run production mutations without approval.

## Operability baseline

| Concern | Required practice |
| --- | --- |
| Service SLO | Define availability/latency/error SLI, SLO window and ownership. Alert on user-impacting error-budget burn with actionable runbook; separate paging from informational alerts. |
| Logs | JSON/structured logs to stdout/stderr with request/trace IDs, version and pod identity; no secrets/PII. Ship off node with retention and access policy. |
| Metrics | Request rate, errors, duration histograms, saturation, queue lag, dependency errors, restarts/OOM, CPU throttling, disk/PVC, probe failures; avoid high-cardinality labels. |
| Traces | Distributed context propagation, sampling budget and service/dependency spans; do not capture sensitive payloads. Correlate traces with metrics/logs. |
| Capacity | Tune requests based on load data and scheduling headroom; memory limits protect nodes, CPU limits may throttle. Account for `emptyDir`, ephemeral storage, quota and node/system reservations. |
| Scaling | HPA/KEDA only with healthy metrics and app-safe scaling. CPU-utilization HPA relies on CPU requests. Stabilize scale-down; bound maximums and downstream capacity. Coordinate VPA with HPA and Pod eviction. |
| Availability | Replicas plus actual zone/node topology-spread/anti-affinity. Readiness/startup/shutdown work together; maintain enough spare capacity for rolling updates and node drains. |
| Disruption | PDB protects against eligible **voluntary evictions**, not Deployment rolling updates, crashes, forced deletions or all outages. Set `minAvailable`/`maxUnavailable` based on tolerable loss. |
| Release | Change one immutable image/config revision; staged canary or progressive rollout where risk warrants; verify metrics after deploy, not just Pod Ready. Keep rollback and schema compatibility. |
| Stateful | Define data owner, access modes, RPO/RTO, backups/snapshots and **regular restore tests**. PVCs and StatefulSets are not backups or replication; handle database failover and fencing separately. |

## Rollouts, upgrades, GitOps

- Prefer declarative GitOps (Argo CD/Flux) for changes. CI should publish an artifact digest; delivery updates the desired state after policy/approval. Do not let manual `kubectl` changes race reconciliation.
- RollingUpdate may run old and new versions together; use compatible expand/contract database migrations, independent migration Jobs and tested backward compatibility. Avoid startup-time migration from every replica.
- Use `maxUnavailable: 0`/`maxSurge: 1` **only** if extra capacity exists, replicas can serve in parallel and readiness is meaningful. For stateful systems, design controller-specific update/failover behavior.
- For high-risk releases, use canary traffic splits, automated metric gates and clear abort conditions; a Kubernetes rollout status alone is not business-health validation.
- GitOps rollback means reverting the desired Git revision/image digest, not only `kubectl rollout undo` (which reconciliation may reverse). Argo CD automatic sync can affect rollback workflows; follow the controller's documented procedures.

## Incident triage (read-only first)

```sh
kubectl get deploy,sts,ds,pods,svc,endpointslices -n "$NS" -o wide
kubectl get events -n "$NS" --sort-by=.lastTimestamp
kubectl describe pod "$POD" -n "$NS"
kubectl logs "$POD" -n "$NS" --all-containers=true --tail=200
kubectl logs "$POD" -n "$NS" -c "$CONTAINER" --previous --tail=200
kubectl top pods -n "$NS"  # requires metrics-server / metrics API
kubectl rollout status deployment/"$DEPLOY" -n "$NS" --timeout=180s
kubectl rollout history deployment/"$DEPLOY" -n "$NS"
```

- **Pending:** capacity/requests, affinity, taints, quota, PVC binding, admission errors. **CrashLoopBackOff:** logs/exit code, wrong entrypoint, config, permissions, OOM. **Unready:** probe mismatch, startup, dependencies, CNI/DNS. **Intermittent latency:** CPU throttling, pools, HPA, queues, DNS, DB saturation. **Volume errors:** CSI, permissions, topology, filesystem, available space.
- Use `kubectl debug` ephemeral containers only when RBAC/PSA and incident policy permit; never assume distroless images have a shell. Prefer debug images from approved registries.
- Do not blindly delete Pods/PVCs, scale stateful components, restart DBs, disable admission, widen firewall rules or run `kubectl rollout undo` during GitOps reconciliation.

## Capacity, scaling and resilience tests

- Test Pod restart/SIGTERM, graceful connection draining, concurrent rollouts, HPA spikes, eviction/node drain, zone failure and dependency loss in a nonproduction or approved test window. Use written fault-scope, owner, abort triggers and recovery steps; avoid chaos tests without bounds.
- Set ResourceQuota, LimitRange, priority and topology policies at platform level. Monitor HPA unable-to-scale, missing metrics, throttling/OOM and cost vs headroom.
- Record RPO/RTO and dependencies; schedule automatic **application-consistent** backups where required. Validate restore into isolated environment, credentials, schema/version and DNS cutover; rehearsed restore matters more than snapshot existence.
- Provide on-call runbooks with dashboards, escalation paths, safe rollback, owner, exact prerequisites and maintenance windows. Record known version incompatibilities and feature gates.

## Deliverable

Provide measured symptoms/baseline, SLO/capacity implications, a minimally disruptive action plan, required approvals, validation evidence, rollback/restore steps, and follow-up tests/alerts. Clearly distinguish observation from hypothesis.

## Official references

- https://kubernetes.io/docs/concepts/cluster-administration/observability/
- https://kubernetes.io/docs/concepts/cluster-administration/logging/
- https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/
- https://kubernetes.io/docs/concepts/workloads/management/
- https://kubernetes.io/docs/tasks/run-application/configure-pdb/
- https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
