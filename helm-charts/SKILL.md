---
name: helm-charts
description: Design, review, refactor, test, and release Helm charts for Kubernetes. Use for Chart.yaml, values.yaml, JSON Schema, Go templates, helper templates, dependencies, CRDs, hooks, chart packaging, release upgrades, and Helm-specific CI validation.
---

# Helm charts

Produce predictable, reusable charts with safe defaults and deterministic rendered manifests. Preserve existing release names, resource ownership, immutable selectors, and persistent data. Do not deploy without authorization.

## Workflow

1. Inspect Helm major version (`helm version`), chart/release ownership, target Kubernetes APIs, dependencies, existing values, downstream overrides, GitOps controller, and live resources if access is approved.
2. Change only what the application needs; compare rendered manifests for every supported environment and test upgrades from the previous chart version.
3. Validate charts and Kubernetes schemas/policies. Report compatibility assumptions and any untested release behavior.

## Chart design

- Use `apiVersion: v2` for application charts, a SemVer `version` for the **chart**, and `appVersion` for the app (metadata only). Increment chart version for published chart changes; never assume `appVersion` pins an image.
- Keep one Kubernetes object per `templates/*.yaml`; put namespaced helpers in `templates/_helpers.tpl`, e.g. `{{ define "mychart.fullname" }}`. Use two-space YAML indentation.
- Keep `values.yaml` small, documented and typed. Use stable maps rather than index-sensitive lists, camelCase keys, and YAML-safe quoting; never quote Kubernetes integers/booleans as strings. **For reusable/production charts, require a `values.schema.json` contract** with types, required nested keys, enum/range/pattern constraints and safe defaults. Schema validates the merged `.Values` (including defaults, `-f`, `--set` and subcharts); use `required`/`fail` only for cross-field rules that JSON Schema cannot express conveniently. Do not set `additionalProperties: false` globally if Helm/subchart overrides need arbitrary keys; scope strictness to owned objects.
- Use `include "mychart.labels" . | nindent 4`, `toYaml | nindent N` for maps/lists, `quote` for strings. Never interpolate arbitrary values as unquoted YAML: values can change structure via newlines.
- Use stable labels (`app.kubernetes.io/*`, `helm.sh/chart`) and small, immutable selectors. Keep release-specific or version-specific labels **out of selectors**. Name resources predictably; avoid collisions across releases.
- Parameterize only meaningful deployment choices (image digest, replicas, resources, probes, service, storage, network) with secure defaults. Don't template every Kubernetes field or silently invent secrets, StorageClasses, URLs or health endpoints.
- Prefer an **existing Secret** or external secret controller. Helm values, rendered templates, release metadata, CI artifacts and `--debug` may expose secrets; do not put plaintext secrets in chart defaults, shell `--set` arguments or Git.
- Lock dependencies in `Chart.yaml` and commit `Chart.lock`. Use `helm dependency build`, not `helm dependency update`, for repeatable CI. Audit third-party charts, repositories and versions; avoid unnecessary umbrella dependencies.
- CRDs in `crds/` are not templated and are **not automatically upgraded or deleted** by Helm. Plan upgrades/migrations separately, especially for operators. Hooks execute outside normal resource lifecycle: use only for unavoidable ordered tasks; make Jobs idempotent with timeout, failure/cleanup and rollback handling. Don't rely on Helm rollback to revert data migrations.
- Avoid `lookup`, random values and timestamps in templates unless explicitly needed: they make offline rendering and GitOps diffs unpredictable. If using a post-renderer, pin it and keep it identical for every upgrade/rollback.

## Minimal patterns

`Chart.yaml`:

```yaml
apiVersion: v2
name: myapp
description: Deploys myapp
version: 0.1.0
appVersion: "1.2.3"
type: application
```

`values.yaml` (illustrative, supply an actual digest through an environment-specific values file):

```yaml
image:
  repository: registry.example.com/team/myapp
  digest: ""  # required at render time
replicaCount: 2
```

`templates/deployment.yaml` fragment:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: myapp
          image: {{ printf "%s@%s" .Values.image.repository (required "image.digest is required" .Values.image.digest) | quote }}
```

Add `values.schema.json` beside `values.yaml` (minimal **real** schema for the fragment above):

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {"type": "integer", "minimum": 0},
    "image": {
      "type": "object",
      "required": ["repository", "digest"],
      "properties": {
        "repository": {"type": "string", "minLength": 1},
        "digest": {"type": "string", "pattern": "^sha256:[a-fA-F0-9]{64}$"}
      }
    }
  },
  "required": ["image", "replicaCount"]
}
```

Do not copy an illustrative empty digest into production: it deliberately fails schema validation. The fragment is not a deployable workload; replace the image and verify the chart's actual selectors and probes.

## Validation and release

```sh
helm version
helm dependency build ./chart
helm lint ./chart --strict -f values/staging.yaml
helm template myapp ./chart -n staging -f values/staging.yaml > rendered.yaml
# Negative test: must FAIL when image.digest is invalid (use valid real values for others):
# helm template myapp ./chart -f values/staging.yaml --set-string image.digest=bad
# Validate rendered.yaml against the target Kubernetes API/CRDs and policies.
# For example, if installed: kubeconform -strict -summary rendered.yaml
# Also render/check *all* supported environment values files.
```

- Test **schema boundaries** with valid/invalid values: missing digest, wrong type, negative replicas, invalid enum/port/range, overridden parent/subchart values. Do not use `--skip-schema-validation` as a shortcut. If public values API changes, test backward compatibility with previous supported values files.
- Compare rendered output against the previous release/chart, including selectors, PVCs, admission policy and resource deletions. Rendering is **not** a cluster test; use authorized server-side dry-run and nonproduction installation when feasible.
- Before an **authorized** upgrade, inspect `helm history`, current values and release ownership. Use version-specific rollback flags: Helm **3** supports `--atomic`; Helm **4** uses `--rollback-on-failure`. Verify options with `helm upgrade --help`. Add `--wait --timeout 10m` only with app-appropriate budgets; `--wait-for-jobs` when applicable. Test the rollback path; it does not roll back database changes.
- If GitOps owns the release, update its chart source/values in Git rather than manually running `helm upgrade` against the same release.
- Never commit rendered manifests containing secrets. Avoid logging `helm get values --all`, release manifests or `--debug` output in public CI logs. Test template injection via special characters/newlines in string values and invalid `tpl` inputs; do not evaluate arbitrary user strings with `tpl` without a documented trust boundary.

## Acceptance

Deliver the full chart change and values contract; show lint, render, schema/policy checks, upgrade risks, chart version change and any runtime evidence. Never claim install/rollback succeeded without a real cluster test.

## Official docs

- https://helm.sh/docs/chart_best_practices/
- https://helm.sh/docs/topics/charts/
- https://helm.sh/docs/chart_template_guide/yaml_techniques/
- https://helm.sh/docs/helm/helm_upgrade/
