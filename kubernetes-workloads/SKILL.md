---
name: kubernetes-workloads
description: Create, refactor, review, and validate deployable Kubernetes workload YAML, Helm charts, and Kustomize overlays. Use for choosing Deployment/StatefulSet/DaemonSet/Job/CronJob, Services and traffic entry, startup/readiness/liveness probes, resource sizing, config and volumes, topology placement, rollout strategies, Helm values, and pre-deploy validation.
---

# Kubernetes workload manifests

Turn a real container's **runtime contract** into minimal, versioned, deployable manifests. Prefer a working, app-specific configuration to a generic "production" template. Never invent endpoints, storage capabilities, image digests, replica counts or resource measurements.

## Procedure

1. Inspect existing deployment method (raw YAML/Helm/Kustomize/GitOps), target Kubernetes version and CRDs, namespaces, admission defaults, runtime contract (image/architecture/UID/ports/health/startup/shutdown), configuration, persistence, expected traffic, node/zone capacity and rollout constraints.
2. Select controller and associated objects. Explicitly record assumptions before creating fields that cannot be deduced. Preserve immutable selectors, existing PVCs, Service identities and resource ownership.
3. Build the **smallest complete set** of manifests for the task; use the patterns below only when their prerequisites hold. Apply the `kubernetes-security` baseline rather than creating conflicting policy exceptions.
4. Render **every environment**; check schemas, API compatibility, unresolved placeholders, policy, immutable fields, image digest and secret exposure. Test scheduling, start, readiness, termination, failure, rollback and persistent data in a nonproduction environment when possible.
5. Return final files/diff, required cluster features, tested assumptions, validation results and an authorized rollout/rollback plan. Do not deploy production changes without approval.

## Controller choice and required decisions

| Controller | Use when | Manifest decisions / common mistakes |
| --- | --- | --- |
| `Deployment` | Replicas are interchangeable; stateless web/API/worker | Set meaningful readiness, replica/rollout budgets, resource requests; old/new versions coexist during rolling update. Avoid single-writer local state. |
| `StatefulSet` | Stable Pod DNS/identity or a **separate** volume per replica | Requires governing `serviceName` (normally headless Service); configure `volumeClaimTemplates`, storage/access/retention and ordered updates. **Does not** replicate or back up data. |
| `Job` | Finite one-shot task or migration/backfill | `restartPolicy: Never` or `OnFailure`; explicit retry/timeout/parallelism and cleanup; expect duplicate execution. Use unique Job name for reruns. |
| `CronJob` | Recurring finite Job | Define timezone, overlap behavior, missed-run deadline, job retries, history. No exactly-once guarantee; `Forbid` can skip runs. |
| `DaemonSet` | Agent on each eligible node | Specify node selection, needed tolerations, requests and safe upgrade; do not copy privileged node-agent settings into apps. |

Use a `Service` only if the workload needs stable in-cluster discovery/traffic; use an approved Gateway API/Ingress implementation for external traffic. Use HPA/PDB only when the app and cluster require them, not as boilerplate.

## Exact workload rules

**Identity and image**
- Use a **verified** `registry/path@sha256:<digest>`; ensure the digest resolves for every target node architecture. Pin image in the workload template, not just CI. Don't mistake `imagePullPolicy: Always` for immutability.
- Keep `app.kubernetes.io/name`, `instance`, `component`, `version` labels consistent; Service/Pod selectors must match. Deployment/StatefulSet `.spec.selector` is immutable: never change it accidentally through chart labels.
- Set `automountServiceAccountToken: false` by default; get the rest of `securityContext`, RBAC, NetworkPolicy, image admission and Secret policy from `kubernetes-security`.

**Service and entry traffic**
- Application must listen on the Pod interface. Named `containerPort` → matching `Service.targetPort`; expose only intended ports. Service `ClusterIP` is usual; avoid `NodePort`/`LoadBalancer` unless needed.
- For HTTP ingress, specify host, TLS, routing and actual `GatewayClass`/`IngressClass`; verify implementation/controller is installed. Gateway/Ingress configuration is not automatically functional without a controller.
- Readiness controls normal endpoint selection; verify EndpointSlices and traffic behavior during shutdown, not merely Pod status.

**Probes and lifecycle**
- `readinessProbe`: cheap, real serving signal; do not use generic TCP success as application readiness without justification.
- Add `startupProbe` when cold start may exceed liveness budget. Its failure window is approximately `failureThreshold × periodSeconds`; tune against worst observed startup.
- Add `livenessProbe` only if a restart can recover a stuck process; **never** restart entire fleets on transient DB/API unavailability. HTTP/TCP probes may use named ports; gRPC probes require a numeric port. `exec` needs an executable, which distroless may lack.
- Choose `terminationGracePeriodSeconds` from max request/job drain + endpoint propagation allowance; SIGTERM handling belongs to `kubernetes-app-design`. Avoid arbitrary `preStop: sleep`.

**Resources and placement**
- Specify CPU/memory **requests** from usage/load tests, plus memory **limit** when appropriate to policy and OOM risk. CPU limit is optional and may throttle; size `ephemeral-storage` for heavy logs/tmp/writes. HPA CPU utilization needs CPU requests.
- `emptyDir.medium: Memory` consumes memory, not disk; limit its size and account for container memory. `fsGroup` may trigger expensive recursive permission fixes; add only if required by CSI/filesystem/UID behavior.
- Use `topologySpreadConstraints` or soft anti-affinity for HA when nodes/zones actually exist. Hard `DoNotSchedule`, node affinity and taints/tolerations may leave Pods Pending. Check available surge capacity and PVC topology first.
- A PDB protects **supported voluntary evictions**, not Deployment rollouts or node failures. Design a PDB so node draining remains possible; `minAvailable` must fit replica count.

**Config and storage**
- ConfigMaps for non-secrets; Secret/external-secret references for credentials. Never embed secret values in chart defaults, manifests, annotations or rendered artifacts committed to Git. Env vars do not live-update; projected volume updates are eventual, and `subPath` does not auto-update. Define a restart/reload strategy.
- `emptyDir` for disposable data; PVC for durable data. Verify StorageClass, access modes (`ReadWriteOnce` is **not** a general multi-node shared volume), expansion, zone binding, reclaim policy, backup/restore and permissions. StatefulSet PVCs usually survive Pod deletion and may survive scale-down/deletion: handle retention explicitly.
- Do not implement schema migration in every app replica/init container. Use an explicit, idempotent, ordered migration Job and compatible expand/contract rollout; Helm hooks can repeat and need reviewed cleanup/rollback behavior.

**Rollout and scaling**
- Deployment `maxUnavailable: 0, maxSurge: 1` preserves desired ready capacity **only with** meaningful readiness and spare capacity; not a universal default. Set `progressDeadlineSeconds`, optionally `minReadySeconds`, according to app warm-up. A failed progress deadline marks rollout stalled; it does **not** automatically roll back.
- Keep historical revisions/rollbacks manageable; GitOps rollback means reverting the desired state, not only `kubectl rollout undo`. Avoid applying manual changes that fight reconciliation.
- StatefulSet `RollingUpdate` updates by ordinal; use partition only for planned staging. Verify PVC data compatibility and the recovery plan before stateful upgrades.

## Starting patterns (syntax checked; runtime untested)

**A. HTTP Deployment + Service.** Assumes `/readyz` exists, port 8080 binds on Pod IP, image supports UID 10001 and writes only to `/tmp`. Replace digest with a real 64-hex SHA-256; resource values, timing and replicas are **illustrative**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels: {app.kubernetes.io/name: api}
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  progressDeadlineSeconds: 300
  template:
    metadata:
      labels: {app.kubernetes.io/name: api}
    spec:
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 45
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile: {type: RuntimeDefault}
      containers:
        - name: api
          image: registry.example.com/team/api@sha256:REPLACE_WITH_REAL_DIGEST
          ports: [{name: http, containerPort: 8080}]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: {drop: [ALL]}
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {memory: 512Mi}
          readinessProbe:
            httpGet: {path: /readyz, port: http}
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          volumeMounts:
            - {name: tmp, mountPath: /tmp}
      volumes:
        - name: tmp
          emptyDir: {sizeLimit: 64Mi}
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  selector: {app.kubernetes.io/name: api}
  ports:
    - {name: http, port: 80, targetPort: http, protocol: TCP}
```

Optional probe fragment, **only if both actual endpoints exist** and restart-on-liveness-failure is safe:

```yaml
# Under containers[]; adapt thresholds to measured startup and failure behavior.
startupProbe:
  httpGet: {path: /healthz, port: http}
  periodSeconds: 2
  failureThreshold: 60
livenessProbe:
  httpGet: {path: /healthz, port: http}
  periodSeconds: 20
  timeoutSeconds: 2
  failureThreshold: 3
```

**B. StatefulSet: governing headless Service + per-replica PVC.** Snippet assumes a real stateful app which stores data at `/data`. Add the actual image, resources, security context, command and app-specific probes to the Pod template. These objects alone are **not deployable** as shown.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector: {app: db}
  ports: [{name: db, port: 5432, targetPort: db}]
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels: {app: db}
  updateStrategy: {type: RollingUpdate}
  template:
    metadata:
      labels: {app: db}
    spec:
      containers:
        - name: db
          image: registry.example.com/team/db@sha256:REPLACE_WITH_REAL_DIGEST
          ports: [{name: db, containerPort: 5432}]
          volumeMounts: [{name: data, mountPath: /data}]
  volumeClaimTemplates:
    - metadata: {name: data}
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: REPLACE_WITH_REAL_STORAGE_CLASS
        resources:
          requests: {storage: 20Gi}
```

Never treat a three-replica StatefulSet as a replicated database by itself; database replication/failover and backup belong to the application/operator and operations design. Do not let a generic Helm upgrade delete/recreate its PVCs.

**C. Finite Job / CronJob.** Run once with bounded retries; never assume exactly-once execution. Examples use hypothetical `/app/task` and must be adapted to a real command and digest.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backfill-v1
spec:
  backoffLimit: 2
  activeDeadlineSeconds: 1800
  ttlSecondsAfterFinished: 86400
  completions: 1
  parallelism: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: backfill
          image: registry.example.com/team/worker@sha256:REPLACE_WITH_REAL_DIGEST
          command: ["/app/task", "backfill"]
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {memory: 512Mi}
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
spec:
  schedule: "0 3 * * *"
  timeZone: Etc/UTC
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 600
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1200
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cleanup
              image: registry.example.com/team/worker@sha256:REPLACE_WITH_REAL_DIGEST
              command: ["/app/task", "cleanup"]
              resources:
                requests: {cpu: 100m, memory: 128Mi}
                limits: {memory: 512Mi}
```

For normal namespaces, add the applicable restricted security context and token rules to Jobs too. Set `podFailurePolicy`, indexed completions and TTL only when task semantics require them. **A TTL-cleaned Job managed continuously by GitOps may be recreated by reconciliation**; design lifecycle/cleanup with the controller. CronJob `timeZone` requires a supported Kubernetes version; test missed schedules and timezone changes.

**D. Optional topology spread, for a multi-zone app with correctly labeled nodes:**

```yaml
# Under Pod spec; soft preference avoids Pending when one zone is unavailable.
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels: {app.kubernetes.io/name: api}
```

## Helm / Kustomize delivery

- Use **one** source of truth per environment; prefer Kustomize for small overlays, Helm when reusable charts/templating are genuinely useful. Follow existing repository conventions; avoid wrapping every YAML field in a value.
- Helm: for reusable charts require `values.schema.json` with typed/required/ranged/enum/pattern constraints and negative validation cases (see `helm-charts`). Chart version tracks chart changes; `appVersion` is metadata, **not** an image pin. Pin chart dependencies, use namespaced helper templates, standard labels, `values.schema.json` for key input validation, and deterministic output. Avoid `lookup`, random values, or secret rendering that changes on each diff.
- Use `required`/schema for genuinely mandatory settings; don't silently invent secrets, StorageClasses or IngressClass. Respect existing release-owned resource names. Do not store credentials in committed values or rendered CI artifacts.
- Kustomize: prefer bases/components plus small environment overlays and image digest overrides; test every overlay. Ensure rendered resource identities/selectors remain stable.
- Config-driven Pod restarts: use a safe config checksum annotation or an explicit rollout update for config that cannot hot-reload; hashing a plaintext Secret into annotations can leak information.

## Validation and acceptance

```sh
# Set NS and use exactly one renderer, with real repository paths.
helm lint ./chart --strict
helm template api ./chart --namespace "$NS" > rendered.yaml
# OR: kubectl kustomize ./overlays/prod > rendered.yaml

# Validate all environments against the actual cluster's Kubernetes/CRD schemas.
# If kubeconform is installed: kubeconform -strict -summary rendered.yaml
kubectl apply --dry-run=client -n "$NS" -f rendered.yaml
kubectl apply --dry-run=server -n "$NS" -f rendered.yaml  # requires API/RBAC/admission
kubectl diff -n "$NS" -f rendered.yaml                   # exit 1 = diff, >1 = error
```

Check: valid image digest (no placeholder), app endpoints and named ports match, selectors/owner references stable, no plaintext secrets, correct namespace and Service traffic, sufficient requests/quota/zone/storage capacity, config update behavior, no immutable field replacements, admission policies, and all target environment renders. **Negative controls:** invalid digest, wrong probe port, missing required config, denied Secret access, unschedulable topology, forbidden `securityContext` and missing storage class should fail at an appropriate policy/admission/runtime gate, not just YAML parsing. `kubectl apply --dry-run` is **not** a smoke test.

After an **authorized** test deployment, check rollout status, events, EndpointSlices and actual service traffic; test SIGTERM, OOM/probe failure, rescheduling and rollback. For Jobs, inspect completion/failure conditions and duplicate execution effects; for StatefulSets, verify stable identity, PVC remount, restart and restore. In GitOps repositories, change Git desired state; do not race its controller with imperative apply.

## Hand-off

Return complete manifests/chart changes, exact prerequisites, justified non-default choices, render/schema/policy and runtime evidence, plus a rollback plan. State untested assumptions; never claim the cluster applied or accepted a manifest merely because YAML parses.

## Primary references

- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- https://kubernetes.io/docs/concepts/workloads/controllers/job/
- https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/
- https://helm.sh/docs/chart_best_practices/
