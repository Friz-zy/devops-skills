---
name: integration-reliability-testing
description: Design and implement deterministic integration, contract, end-to-end, performance and resilience tests for applications and distributed systems. Use for Testcontainers, real dependencies, retries/timeouts, failover, idempotency, concurrency, fault injection, load thresholds and release gates.
---

# Integration and reliability testing

Test **observable invariants and failure behavior**, not merely green happy paths. Make test targets isolated, bounded and repeatable; do not inject faults into production without written authorization, stop rules and recovery ownership.

## Workflow

1. Identify critical user journeys, external dependencies, API/event contracts, consistency model, SLOs, expected concurrency/load and failure modes. Write measurable acceptance conditions before tools.
2. Build deterministic fixtures with pinned versions of real dependencies (Testcontainers, ephemeral DB/queue, local Kubernetes when needed); mock only uncontrollable external services at clear boundaries.
3. Add normal + failure cases, concurrency and recovery checks. Set test timeouts, cleanup, resource/port isolation and secrets redaction; ensure tests cannot reach production credentials/endpoints.
4. Run locally/CI on supported platforms, reproduce failures, collect structured evidence; track flaky tests and do not use broad retries to hide them.

## Test layers and contracts

- Unit tests for pure logic; integration tests for DB schema, transactions, serialization, message delivery, auth and actual service dependencies. Contract tests for producer/consumer schema and version compatibility; E2E only for a small set of critical workflows.
- Use pinned container images, a fresh database/schema/namespace per test, deterministic seeds and synthetic non-PII data. Restore state between tests; never share writable test resources across concurrent jobs without isolation.
- Test HTTP/gRPC/API compatibility, auth-denied paths, schema evolution, pagination and limits, and error responses. For asynchronous flows verify duplicate/reordered/delayed delivery, poison messages, DLQ, retries, exactly-once **illusion**, checkpointing and idempotency keys.
- Exercise transaction rollback, retry-safe writes, concurrent updates/optimistic locking, eventual consistency and cross-service partial failure. Assert data invariants (no loss, duplication or unauthorized access), not only response codes.

## Resilience boundaries

- Every external call needs explicit deadline/timeout and cancellation propagation; limit retries by count/total time with exponential backoff + jitter **only** for retryable/idempotent operations. Do not retry the entire request at multiple layers without a shared budget.
- Bound worker/concurrency pools, queues, memory and connection limits. Test backpressure, overload shedding, rate limiting, circuit breakers where appropriate and graceful shutdown under in-flight traffic.
- Fault matrix: dependency timeout/reset, DNS failure, 429/5xx, DB restart/failover, queue lag/duplicate delivery, process SIGTERM/OOM, network partition/latency and disk-full only where safe. Verify recovery and no residual test resources.
- Load/stress/soak tests: document hardware, data shape, concurrency and target p50/p95/p99, throughput, saturation and errors. Compare to baseline and SLO/error budget; performance tests must not perturb shared production workloads.
- Run chaos experiments only in a named isolated scope with max duration/blast radius, owner, observability, abort threshold and rollback. No uncontrolled `kill -9`/network disruption on shared infrastructure.

## Concrete acceptance checks

```sh
# Run the repository's actual unit/integration suites; examples only:
go test -race ./...                    # Go, supported architectures
pytest -q tests/integration           # Python, only if that suite exists
# Optional tooling, if configured: Testcontainers, Pact/Spring Cloud Contract,
# k6/Locust, Toxiproxy, kind, fault injection through controlled test harnesses.
```

- Assert time-bounded completion, clean resource teardown, reproducible failure IDs/seeds and version coverage. Test retry storms with a failing downstream and verify bounded traffic.
- Test incompatible schema/API version and unauthorized access as **negative controls**. Contract gates must detect a deliberately broken provider/consumer fixture.
- Check concurrent reruns and CI parallelism; distinguish flaky infrastructure from a functional regression. Don't claim a test is hermetic if it depends on public network or mutable images.

## Handoff

Provide test matrix and fixtures, required tools/setup, exact assertions and thresholds, CI gating levels (fast PR vs scheduled load/chaos), execution evidence, known uncovered risks and cleanup steps.

## Sources

- https://sre.google/workbook/table-of-contents/
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- https://docs.docker.com/guides/testcontainers-cloud/
- https://go.dev/doc/tutorial/fuzz
