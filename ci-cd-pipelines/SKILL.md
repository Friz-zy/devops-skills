---
name: ci-cd-pipelines
description: Design, implement, review, or fix secure and efficient CI/CD pipelines in GitHub Actions, GitLab CI, or other runners. Use for test/build/release workflows, conditional jobs, caching, artifacts, container publication, security scans, SBOM, provenance, signing, and controlled deployment.
---

# CI/CD pipeline engineering

Design a working, maintainable pipeline for the **actual repository and runner**. Prefer simple, fast feedback; isolate untrusted code from deployment credentials. Do not assume any CLI, scanner, runner permission, or external service is already installed.

## Workflow

1. Inspect repository, CI provider/version, branch protection, languages, lockfiles, required tests, runner OS/architecture, release destinations, deployment owner (direct CI or GitOps), secret sources, and existing workflows.
2. Draw the job dependency graph and trust boundary. Separate **untrusted PR validation** from **trusted release/deploy**; determine required tools, versions, identities, network access, and permissions.
3. Implement the smallest working pipeline, pin third-party code/images, install or invoke each required scanner explicitly, and verify its version/checksum. Use reusable jobs only when duplication justifies them.
4. Validate pipeline syntax, test all event/rule paths, check PR isolation and artifact flow, then run on an eligible runner. Report missing tools, failed checks, and any untested deploy path.

## Pipeline stages and gates

| Stage | Minimum contract |
| --- | --- |
| Trigger | PR/MR: lint, tests, safe scans, no publish/deploy credentials. Protected branch/tag: release. Scheduled: dependency/base-image rescans. Manual: explicit approved recovery/promotions. |
| Fast checks | Format, lint, typecheck, dependency/lockfile integrity, unit tests, IaC and YAML schema checks where relevant. Fail required checks; don't hide errors behind `|| true`. |
| Test | Integration/contract tests with isolated ephemeral services, deterministic fixtures, timeouts; test matrix only for supported OS/runtime/architecture. Collect JUnit/coverage on failures. |
| Build | Produce one immutable artifact from reviewed commit; preserve source revision, build parameters, toolchain versions and artifact digest. Never rebuild per environment. |
| Security | Secret scanning, dependency/SCA, IaC checks, image scan after build, optional SAST/DAST and license policy. Explicitly provision tools and update their DBs; policy needs defined severity, fixes, triage, expiry. |
| Supply chain | Generate SBOM (SPDX/CycloneDX) and provenance tied to the digest. Sign with verified CI identity when required; **verify** signer/issuer/subject. Unsigned BuildKit attestations are not equivalent to verified signatures. |
| Release | Publish only from protected refs and isolated jobs with narrow OIDC/short-lived permissions. Record image/binary digest, SBOM, provenance, changelog and checks. |
| Deploy | Prefer a reviewed GitOps commit updating digest; otherwise restrict cluster credentials and use protected environments/approvals. Promote **same artifact**, preserve OCI referrers, verify destination and health; provide rollback. |

## Security boundaries

- Default credentials to read-only; grant write, `id-token`, registry and environment access only to the job needing them. Use OIDC/workload identity; avoid static cloud keys and PATs.
- Treat fork PRs, PR titles/branches, issue text, artifacts, caches, dependencies and scripts as untrusted. Never execute PR-controlled code in a privileged `pull_request_target`/equivalent context or interpolate untrusted data into shell scripts. Checkout the expected ref explicitly.
- Prefer GitHub workflow dependency lockfiles where supported; otherwise pin actions to **reviewed full commit SHAs** and refresh via Renovate/Dependabot. Pin GitLab includes/components and job images to reviewed revisions/digests. Audit third-party plugins, workflow dependencies and external downloads.
- Prefer ephemeral, isolated runners for untrusted code. Avoid privileged Docker-in-Docker, shared daemon socket, long-lived self-hosted runners and shared writable caches across trust boundaries; otherwise document the security design.
- Treat cached dependencies as untrusted accelerators, not correctness inputs. Key by lockfile/platform/tool version and separate PR caches from release caches. Never cache secret-bearing files or inject credentials into build args/history.
- Protect default branch/tags, require reviews and status checks, use environment approvals, protected variables and concurrency controls. Enforce security gates in an externally protected required workflow or ruleset where possible; a PR author must not be able to remove required checks simply by editing pipeline YAML. Do not let mutable tags or external artifacts bypass test gates.

## Provider-specific implementation

**GitHub Actions:** use minimal workflow/job `permissions`, `concurrency` (cancel superseded PR runs, not production releases), `timeout-minutes`, explicit environments, OIDC `id-token: write` only in identity-consuming jobs. Pass artifacts through immutable digests; verify downloaded artifacts/attestations. Use `actionlint` for workflow static checks. Example reusable fragments:

```yaml
permissions:
  contents: read
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

**GitLab CI:** use `workflow:rules` to avoid duplicate push/MR pipelines, `rules` for job conditions, `needs` for DAG dependencies, `artifacts:reports` for tests, `resource_group` for serialized deploys, protected environments/variables, and `id_tokens` for OIDC. Test with GitLab CI Lint. If a `needs` job may be excluded by rules, align rules or set `optional: true` where correct. Example triggers:

```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

**Other CI systems:** apply the same privilege separation, immutable job dependencies, reliable DAG, artifact identity and deploy authorization; do not blindly translate provider semantics.

## Container release pattern (when images are produced)

- Ensure Buildx/Podman/Buildah, scanners and signing clients are installed **in the job or pinned job image**. Prefer registry-backed `--push` for multi-platform images/attestations; classic Docker `--load` may drop attestations.
- Example **after** verifying Buildx is available and registry authentication is configured (replace variables with real values):

```sh
set -eu
: "${IMAGE:?registry/repository required}"
: "${VERSION:?immutable release version required}"
docker buildx build --check .
docker buildx build --pull --platform linux/amd64,linux/arm64 \
  --sbom=true --provenance=mode=min --metadata-file build-meta.json \
  -t "$IMAGE:$VERSION" --push .
DIGEST="$(jq -er '."containerimage.digest"' build-meta.json)"
printf '%s@%s\n' "$IMAGE" "$DIGEST"
```

- Scan **published digest**, not only source; run smoke tests against the exact released artifact on supported platforms. Set scan DB freshness and fail criteria; do not treat a zero-finding result as proof of safety.
- Sign OCI digest with trusted CI identity (`cosign`/Notation when configured), then verify certificate identity/issuer and attestations **before** promotion. A registry copy may change digests or lose referrers: copy/verify images and associated attestations together.
- Do not claim a particular SLSA level based only on enabling `--provenance`; the build platform's isolation and attestation trust must satisfy that level.

## Boundary and test matrix

- Verify each event in a **trigger matrix** (fork PR, internal PR, default branch, release tag, manual, schedule). In an untrusted PR, assert cloud OIDC/prod secrets/push credentials are unavailable; ensure fork-controlled code cannot run with privileged `pull_request_target` or trigger a trusted artifact download without verification.
- Test pipeline linter + job DAG and required-status checks. Include a **deliberately failing** lint/test/security fixture to prove the gate blocks promotion. Compare artifact digest across build, scan, sign, registry copy, deployment and rollback.
- Tools are explicit dependencies: pin installer action/container binary, verify checksums/provenance where supported, configure scanner DB caching/freshness and fail conditions; do not list a tool without wiring an executable job.
- Ansible/Terraform/Kubernetes deployment jobs need separate narrowly scoped identities. CI validation of an untrusted PR must not run Terraform data sources or cloud plans with production write credentials.

## Performance and reliability

- Fast checks first; run independent jobs in parallel with `needs`/DAG. Cache safely, deduplicate identical pipelines, avoid broad monorepo rebuilds using dependency-aware paths; never skip critical release checks.
- Apply job timeouts, limited retries only for transient external errors, artifact retention, concurrency rules, immutable promotion and clear manual rollback. Do not retry failing tests indiscriminately or make deploy steps race.
- Validate shell steps with strict error handling where compatible; quote variables, minimize `curl | sh`, verify downloads, avoid leaking tokens via debug output.

## Deliverable / acceptance

Return CI files or a patch; dependency graph; tool installation/pinned versions; permissions/secrets; trigger matrix; artifacts and digest flow; tests/security gates; deploy/rollback plan; executed validation and explicit blockers. No "secure" or "passing" claim without evidence.

## Official references

- https://docs.github.com/en/actions/how-tos/secure-your-work
- https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments
- https://docs.gitlab.com/ci/pipeline_security/
- https://docs.gitlab.com/ci/yaml/
- https://slsa.dev/spec/v1.2/build-track-basics
- https://docs.docker.com/build/ci/github-actions/attestations/
