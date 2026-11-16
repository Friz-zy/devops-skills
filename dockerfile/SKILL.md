---
name: dockerfile
description: Create, review, fix, and optimize Dockerfiles/Containerfiles and .dockerignore files. Use for base images, deterministic dependencies, multi-stage builds, BuildKit caching and secrets, image size, non-root users, file ownership, entrypoints, and Dockerfile security.
---

# Dockerfile engineering

Produce a small, secure, reproducible **runtime image that actually works**. Follow repository conventions and preserve behavior; do not apply hardening blindly.

## Workflow

1. Inspect the Dockerfile, `.dockerignore`, lockfiles, build context, supported architectures, language runtime, native dependencies, expected UID, ports, writable paths, and startup/shutdown behavior.
2. Identify risks: untrusted/stale base, missing lock, bloated final image, secret leakage, cache misses, incompatible libc/ABI, permissions, signals, and multi-platform breakage.
3. Make the smallest correct changes; use the patterns below **only where applicable**. Avoid breaking CA certificates, time zones, locale, DNS, file access, debugging, and target architecture.
4. Validate build, execution, shutdown, permissions and image content when tools are available. Report what was and was not executed.

## Rules by concern

| Concern | Required practice |
| --- | --- |
| Base | Use a trusted, supported, minimal **compatible** image with explicit runtime version and OS variant. Pin `@sha256` for immutable builds; automate digest refreshes. `slim` is often safer for compatibility than Alpine (`musl`). Use distroless/scratch only when runtime dependencies and debugging are understood. |
| Builds | Prefer multi-stage when compilers, test tools or dependency managers are not needed at runtime. Keep only required files in the final stage. Match architecture and libc between builder and runtime. |
| Packages | Use lockfiles and deterministic installers (`npm ci`, `--frozen-lockfile`, Go checksum validation, etc.). Combine `apt-get update` with `apt-get install -y --no-install-recommends` in one `RUN`, then remove apt lists. Avoid broad `apt-get upgrade`; rebuild with updated bases. |
| Context | Use `.dockerignore` to exclude `.git`, `.env*`, secrets, caches, build artifacts and irrelevant files; don't exclude required inputs. Prefer specific `COPY`. Use `ADD` only when its special features are intentional. |
| Cache | Put stable dependency manifests before volatile source. Use BuildKit cache mounts; they must not be required for correctness. Be careful with shared/external caches and untrusted PRs. |
| Secrets | Use `RUN --mount=type=secret` or SSH mounts for private dependencies. Do **not** put credentials in `ARG`, `ENV`, files copied to layers, logs, or provenance. Do not blindly execute `curl | sh`; verify source and checksum/signature. |
| Identity | Prefer a nonzero numeric UID/GID; use `COPY --chown`/`--chmod` and minimum read/write permissions. Support read-only root filesystem via explicit writable paths. Some platforms require arbitrary UIDs; verify compatibility. |
| Process | Use exec-form `ENTRYPOINT`/`CMD`, handle SIGTERM, and ensure PID 1 reaps children if needed. Shell wrappers must `exec "$@"`; use `tini` only if the process needs it. Avoid background daemons and unnecessary multi-process containers. |
| Metadata | Include accurate OCI `source`, `revision`, `version`, `licenses` when known. `EXPOSE` documents a port; it does not publish it. Docker `HEALTHCHECK` is for Docker/Compose consumers; Kubernetes uses Pod probes. |

### Debian/Ubuntu package pattern

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

### Secrets and dependency cache (BuildKit)

```dockerfile
# syntax=docker/dockerfile:1
COPY package.json package-lock.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc,required=true \
    --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
```

```sh
docker buildx build --secret id=npmrc,src="$HOME/.npmrc" -t app:test .
```

Never use `ARG TOKEN`, `ENV TOKEN=...`, or `RUN echo "$TOKEN"` for secrets. Do not copy source or package credentials before the secret mount is available. Set correct permissions and user for mounted credentials if the `RUN` step is non-root.

### Example: cross-platform static Go service

Illustrative versions: choose currently supported versions and resolve production base images to verified digests. This example assumes a valid `go.sum`, static `CGO_ENABLED=0` build, and a `/healthz`-independent binary.

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.26.11-bookworm@sha256:dc40a13b61304647ed7e5d581d246b86413dfc29d63c6d018310feb93cd98fc0 AS build
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -trimpath -ldflags='-s -w' -o /out/app ./cmd/app


FROM gcr.io/distroless/static-debian13:nonroot@sha256:25ac8f0c3b2334a9eb05452266bd4697864a8c8ef0ef71e5e6480d7f92868329
COPY --from=build --chown=65532:65532 /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

- Check the binary is really static (`CGO_ENABLED=0` may not suit every project); static images cannot execute a shell. Distroless base images may include CA certificates, but confirm any required zones/native libraries.
- If using a shell startup script, start with `#!/bin/sh`, handle signals and use `exec "$@"` for the final process.
- Node: use a locked install (`npm ci`), compile with dev dependencies in builder, install/copy **production-only** dependencies to runtime; test native module ABI compatibility.
- Python: use a lockfile and compatible wheel ABI; virtual environments copied across stages must keep the same paths, Python version and OS ABI. Test runtime imports.
- Java: use an appropriate JRE; Rust/Go: check static vs dynamic linking, certificates and DNS.

### Minimal `.dockerignore` pattern

```gitignore
.git
.env
.env.*
*.pem
*.key
node_modules
.venv
__pycache__
.pytest_cache
dist
coverage
```

Add application-specific exclusions; confirm no required generated/hidden files are excluded. Also ensure secrets are not reachable via alternative build contexts or `COPY --from`.

## Validation (execute when available)

```sh
hadolint Dockerfile                       # optional
# Docker Buildx check uses the default Dockerfile in this context.
docker buildx build --check .
docker buildx build --pull --progress=plain --load -t app:review .  # one host platform

docker image history --no-trunc app:review
```

- Smoke-test the **real** entrypoint, user, working directory, mounts, runtime imports, HTTP bind address, and SIGTERM within intended grace period.
- Test with default and explicitly assigned **arbitrary non-root UID** if the target platform requires random UIDs.
- Test a negative build input (missing/invalid lockfile or missing required secret) fails rather than silently continuing.
- Test `--read-only` and explicit tmpfs/volumes; inspect whether the app writes to `/root`, `/app`, or language caches unexpectedly.
- Verify target architectures separately; `--load` is not a portable way to export multi-platform attestation images to the classic Docker image store.
- Look for secrets in final filesystem, history, image config, and CI logs. Scanners alone cannot prove no leakage.


## Output contract

Return the complete Dockerfile and `.dockerignore` (or a focused patch), assumptions and compatibility risks, and validation evidence. Do not report builds/tests as passed unless actually executed.

## Official references

- https://docs.docker.com/build/building/best-practices/
- https://docs.docker.com/build/cache/optimize/
- https://docs.docker.com/build/building/secrets/
- https://docs.docker.com/reference/dockerfile/
