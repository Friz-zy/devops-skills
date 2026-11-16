---
name: kubernetes-app-design
description: Design, write, or refactor application code to run reliably as a Kubernetes workload. Use for health endpoints, startup and SIGTERM shutdown, config and secrets interfaces, statelessness, idempotency, safe concurrency, retries, background workers, migrations, telemetry, and runtime compatibility.
---

# Kubernetes-ready application design

Implement correct **application behavior** before adding Pod YAML. Kubernetes restarts and reschedules processes; it does not make an unsafe application reliable or exactly-once.

## Workflow

1. Inspect runtime, HTTP/gRPC/queue entrypoints, dependency graph, external state, startup time, process signals, concurrency, request/job durations, data migrations, config/secrets, and shutdown requirements.
2. Define observable contracts for startup, readiness, liveness, draining, metrics and safe termination. Keep business logic independent of Kubernetes APIs unless explicitly needed.
3. Implement bounded timeouts, retries, concurrency and idempotency; make containers runnable by non-root arbitrary UID, with read-only rootfs and explicit scratch/data mounts where feasible.
4. Test lifecycle, overload, failure injection, duplicate delivery, dependency loss, restart, and clean shutdown. Document required ports, volumes, probes, resources, config and graceful termination period for the workload skill.

## Required application contracts

| Concern | Implementation practice |
| --- | --- |
| Process | Run one main foreground process; use exec-form entrypoint; listen on `0.0.0.0` or the intended Pod interface, not `127.0.0.1` for Service traffic. Do not daemonize; handle SIGTERM and reap subprocesses if spawned. |
| Startup | Validate essential config early; initialize required resources; fail fast on unrecoverable errors. Avoid long blocking init where parallel/async is possible. Migrations should not be blindly executed by every replica. |
| Readiness | Expose cheap `/readyz` (or gRPC health); return success only while able to accept **new** work. Mark not-ready during drain. Check critical prerequisites without turning transient downstream glitches into fleet-wide outages. |
| Liveness | Optional `/healthz` should test internal deadlock/irrecoverable health, **not** dependence on external DB/API. An indiscriminate restart loop can amplify outages. |
| Startup probe | Needed for cold starts longer than liveness tolerance; expose a real endpoint and document worst-case boot/warm-up time. Kubernetes configures probes separately. |
| Shutdown | On SIGTERM, stop accepting work, set readiness false, drain in-flight requests/workers, release leases, close connections, flush buffers and exit within grace timeout. Account for endpoint propagation and load balancer behavior. |
| Statelessness | Keep sessions/state in external stores; put temporary files in `/tmp` or declared volumes. Never rely on Pod identity, local filesystem durability, fixed IP, or one replica. |
| Work delivery | Queue handlers and scheduled jobs must tolerate retries/duplicates; use idempotency keys, transactions, checkpoints and sensible acknowledgement semantics. Kubernetes Jobs do **not** guarantee exactly-once. |
| Networking | Set client/connect/read/write timeouts; cancel work on request context; use bounded retries with jitter/backoff, retry budgets and circuit breaking. Avoid unbounded queues and connection pools. |
| Scale | Avoid process-global mutable coordination; use external locking/leader election only if required. Limit concurrency and protect downstream services against sudden replica scaling. |
| Config | Separate image from environment; validate typed config, allow safe reload or restart; mount secrets as files when practical, never log them. Document required/recommended/optional variables and defaults. |
| Resource use | Bound heap/cache, goroutines/threads, open files, queue size and ephemeral storage; load-test cold start, latency, OOM behavior, CPU throttling and cancellation. |
| Telemetry | Structured logs to stdout/stderr, useful RED metrics, trace/context propagation (W3C where supported), correlation IDs and bounded-cardinality labels. Never log tokens, credentials or sensitive payloads. |
| Logs | JSON/structured logs to stdout/stderr with request/trace IDs, version and pod identity; no secrets/PII. Ship off node with retention and access policy. |
| Metrics | Request rate, errors, duration histograms, saturation, queue lag, dependency errors, restarts/OOM, CPU throttling, disk/PVC, probe failures; avoid high-cardinality labels. |
| Traces | Distributed context propagation, sampling budget and service/dependency spans; do not capture sensitive payloads. Correlate traces with metrics/logs. |

## Lifecycle example (pseudocode; adapt to language/framework)

```text
start:
  load and validate configuration
  start dependencies and HTTP/gRPC listener
  signal ready only when requests can be served
on SIGTERM:
  mark not ready; stop accepting new work
  drain active requests/jobs within a deadline
  flush telemetry; close clients; exit 0
on fatal error:
  log cause (no secrets); exit nonzero
```

Do not sleep for a fixed period or rely only on `preStop` to drain. Readiness changes and removal from routing are not instantaneous; test real termination under load and set a suitable grace period.

## Language/runtime checks

- **Go:** `signal.NotifyContext`, `http.Server.Shutdown(ctx)` with a deadline; don't close shared resources before handlers complete. Check goroutine/connection leaks.
- **Node.js:** handle `SIGTERM`, stop `http.Server` accepting new connections, end background work and close pools; don't call `process.exit()` before pending work finishes.
- **Python:** ensure ASGI/WSGI worker manager forwards signals; tune worker count vs memory and avoid unbounded background tasks. Use server shutdown hooks.
- **Java:** verify JVM container memory/CPU awareness, graceful web-server shutdown and signal propagation; measure GC pause and cold-start effects.

## Acceptance tests

- Starts with non-root UID and read-only rootfs plus only documented writable mounts.
- Health endpoints respond as specified; cannot mislead during startup/drain or transient dependencies.
- Sends correct exit code on fatal configuration errors; responds to SIGTERM before configured grace timeout.
- Multiple replicas work; duplicate queue messages / retries / node restarts do not corrupt data. Test a forced mid-transaction crash and idempotent replay, and make a duplicate request return the original result when semantics require it.
- Saturation, dependency loss, and rolling old/new-version overlap are tested; include DNS timeout, 429/503, connection reset, slow queue, no writable home, SIGTERM under concurrent traffic and app losing readiness while liveness remains healthy; database schema changes are backward-compatible (expand/contract).
- Logs/metrics/traces expose necessary signals without secret leakage or unbounded cardinality.

## Output

Provide code changes/tests and a **runtime contract**: command, port/bind, non-root UID policy, writable mounts, endpoints/protocols, startup/grace timings, config/secret list, CPU/memory estimate and telemetry. Mark untested behavior.

## References

- https://12factor.net/
- https://kubernetes.io/docs/concepts/workloads/pods/probes/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-termination/
- https://kubernetes.io/docs/concepts/cluster-administration/logging/
- https://opentelemetry.io/docs/concepts/
