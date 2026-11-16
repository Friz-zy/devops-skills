---
name: database-migrations
description: Design, review, test and execute safe database schema/data migrations for relational applications. Use for zero-downtime expand/contract, SQL DDL locks, index creation, backfills, migration tools, multi-version application compatibility and recovery.
---

# Safe database migrations

Prioritize data integrity, bounded lock time, application compatibility and a known recovery path. Never run migrations on production or modify real data without explicit approval and verified target identity.

## Workflow

1. Identify DB engine/version, migration tool/version, table size/traffic, indexes/constraints, replication, transaction semantics, current schema, oldest active app version, backup/restore capability and deployment order.
2. Classify operation: additive schema, concurrent index, constraint, type change, rename/drop, backfill or cleanup. Estimate lock, rewrite, WAL/redo, replicas/lag and disk impact using real schema and data size.
3. Design **expand → migrate/backfill → switch reads/writes → contract** across deployments; all overlapping app versions must work with intermediate schemas.
4. Test on representative restored data and load; run approved migration in a single orchestrated job, monitor health/locks/lag, verify counts and invariants, and retain recovery steps.

## Non-negotiable rules

- Migrations must be version-controlled, ordered, uniquely identified and immutable after release (Flyway/Liquibase/framework native). `validate` checksum history; do not edit an already-applied migration to hide failures.
- Never run mutating migrations from every app replica or startup/init container. Use one coordinated migration job with access/time bounds and clear retry semantics; document framework-specific transaction behavior.
- Add nullable columns/defaults and new tables before deploying readers/writers. Avoid one-release column rename/drop, incompatible NOT NULL constraints and type rewrites while old versions still run. Deprecate reads/writes first; contract only after fleet/queues/jobs are drained.
- Avoid long transactions and locks on hot tables. Use **engine/version-specific** online DDL: PostgreSQL `CREATE INDEX CONCURRENTLY` cannot run inside a transaction and may leave INVALID indexes; MySQL online DDL algorithms still take metadata locks. Do not assume every `ADD COLUMN`/`ALTER TABLE` is non-blocking. On PostgreSQL 11+, non-volatile defaults (including `now()`, which is STABLE) normally avoid full table rewrites, but volatile defaults such as `clock_timestamp()` do not; even fast DDL may still need a strong table lock.
- Set/verify bounded `lock_timeout` and `statement_timeout` (PostgreSQL) appropriate to each operation, schedule low-risk windows, assess index/constraint validation, disk and replication lag; fail safely rather than wait behind locks indefinitely.
- Backfill in **small, deterministic, restartable batches** by stable key with checkpoint/progress, rate limit and idempotent upsert/update semantics. Observe row count, latency, dead tuples/bloat, replica lag and error budget. Do not hold a transaction for an entire large table.
- Protect integrity with unique/FK/check constraints when feasible; validate existing data before enabling strict validation. Feature flags and dual-write/read transitions must handle concurrent updates and retries; test no lost/duplicated writes.
- Rollback of destructive DDL/data is rarely safe. Prefer **roll forward** with app-compatible schema and tested restore/PITR fallback; review backup freshness and RPO/RTO. Do not claim that rolling back an app deployment rolls back its database.
- Least privilege: migration principal separate from app principal, short-lived access, restricted schema permissions; do not log URLs/passwords, SQL parameter values, dumps or table samples containing secrets.

## Validation gates

- Review generated SQL, framework transactions and per-table lock modes; inspect representative `EXPLAIN` (with safe execution) and catalog statistics. Do not run `EXPLAIN ANALYZE` for mutating SQL on production merely to estimate runtime.
- Test empty DB upgrade, production-like schema/data upgrade, retry of failed migration, migration checksum validation, and concurrent old/new app operation; cover rollback to old app with **expanded** schema.
- Perform negative cases: blocked lock, replica lag, partial backfill, duplicate events, missing column, mid-step process kill, out-of-disk/timeout. Assert rows/constraints/invariants before and after, not only exit code.
- For PostgreSQL after concurrent index: confirm `pg_index.indisvalid` and absence of leftover invalid index. Compare data counts/hashes where appropriate; monitor locks, write latency and replication.
- Explicitly document which checks cannot be run without a DB copy or credential. Never use a destructive migration to test its own safety.

## Handoff

Provide exact ordered migration files, compatible app versions, predicted DDL locks/data impact, privilege needs, backfill instrumentation, dry-run/test evidence, abort thresholds and forward recovery/restore plan.

## Sources

- https://docs.gitlab.com/development/database/avoiding_downtime_in_migrations/
- https://docs.gitlab.com/development/database/batched_background_migrations/
- https://www.postgresql.org/docs/current/sql-createindex.html
- https://www.postgresql.org/docs/current/explicit-locking.html
- https://www.postgresql.org/docs/current/ddl-alter.html
- Practical case studies are secondary evidence: verify all locking/volatility claims against the DB engine documentation.
