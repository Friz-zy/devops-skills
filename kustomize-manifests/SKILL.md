---
name: kustomize-manifests
description: Build, refactor, review, and validate Kubernetes manifests using Kustomize bases, overlays, components, patches, image digests, ConfigMap generators, and replacements. Use for kustomization.yaml, environment overlays, kubectl kustomize, GitOps-ready YAML and drift-safe declarative changes.
---

# Kustomize manifests

Prefer plain Kubernetes YAML plus minimal, explicit environment differences. Make every overlay render independently without accidental renames, selector changes or data loss. Do not apply production changes without authorization.

## Workflow

1. Inspect the Kustomize version (`kustomize version` or `kubectl version --client`), base/overlays, GitOps ownership, target API/CRDs, namespaces, selectors, storage, image references and all target environments.
2. Put common resources in `base/`; add only real differences in `overlays/<environment>/`. Use built-in transformers and small patches instead of copying entire manifests.
3. Render every overlay, compare with prior desired/live state, validate against target schema/policies and test a safe deployment when authorized.

## Rules

- Use `resources:` to compose bases and resources. Prefer local bases; pin remote Git refs to immutable commits and review sources. Avoid remote bases that silently change under an existing ref.
- Use `patches:` with explicit `target` or resource identity; use strategic merge for built-in fields and JSON 6902 when exact array/index operations or CRD schema make it safer. Avoid deprecated `patchesStrategicMerge`, `patchesJson6902`, `bases` and `vars` in new configurations; prefer `replacements` when copying generated names/fields.
- Keep selectors stable: changing Deployment/StatefulSet selectors can require replacement. Prefer `labels:` with intentional `includeSelectors` behavior; avoid broad `commonLabels` transformations that modify selectors. Changes to `namePrefix`, `nameSuffix`, `namespace` or naming rules can orphan resources or cause PVC/Service identity changes.
- Use `images:` to update images by **digest**, not fragile text replacement. Confirm platform coverage and validate the actual digest; never use moving production tags.
- Use `configMapGenerator` for non-secret application config. Name hash suffixes enable new references/rollouts when content changes; do not disable hashes casually. Check name-reference updates and rollout behavior for CRDs/custom fields. `secretGenerator` is **not encryption**: never commit plaintext secret literals or files; prefer an external-secret controller or encrypted GitOps workflow.
- Use `replacements` for field propagation; avoid environment variables, custom scripts and Helm templating unless a native Kustomize feature cannot solve the problem. Avoid unnecessary custom transformers/plugins that make builds nonportable.
- Keep one owner for each resource; do not mix `kubectl apply`, Helm and GitOps on the same object without a deliberate management boundary. Treat deletions, renames, `prune` and GitOps automated pruning as potentially destructive, especially for PVCs/CRDs.
- Pin and test the CLI version: standalone `kustomize` and the version embedded in `kubectl` may render differently. Test generated YAML for each supported cluster version, including CRDs and admission rules.

## Minimal structure

```text
k8s/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
  overlays/
    staging/
      kustomization.yaml
      replicas.yaml
    production/
      kustomization.yaml
      replicas.yaml
```

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

`overlays/production/kustomization.yaml` (replace digest with a verified value):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: registry.example.com/team/myapp
    digest: sha256:REPLACE_WITH_REAL_DIGEST
patches:
  - path: replicas.yaml
    target:
      kind: Deployment
      name: myapp
```

`overlays/production/replicas.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
```

Assumes the base defines a Deployment `myapp` and uses exactly `registry.example.com/team/myapp`. This is a pattern, not a deployable app by itself.

## Validation

```sh
kubectl kustomize k8s/overlays/staging > staging.yaml
kubectl kustomize k8s/overlays/production > production.yaml
# If installed: kubeconform -strict -summary production.yaml
kubectl apply --dry-run=client -f production.yaml
# If cluster access/RBAC is approved:
kubectl apply --dry-run=server -f production.yaml
kubectl diff -f production.yaml  # exit 1 = differences; >1 = error
```

- Run `kustomize build` instead of `kubectl kustomize` only when deliberately using the standalone pinned version; do not assume they are identical.
- Check **every** resource identity, namespace, immutable selector, generated reference, image digest, PVC/CRD diff, unresolved placeholder, Secret exposure, and policy result. Compare machine-readable resource identity sets (`apiVersion`, `kind`, `metadata.namespace`, `metadata.name`) across before/after renders; fail on unexpected deletes/renames or duplicate objects.
- Test **negative controls**: deliberately wrong image digest, patch target not matching any resource, invalid API field, unapproved namespace/selector change, plaintext Secret generator input. A successful build alone is not proof a patch hit its intended target; inspect rendered fields.
- If a base contains a CRD, check schema-aware transformation/name references and version-specific behavior. In GitOps, verify controller rendering version matches local pin. Do not rely on YAML parsing alone.
- GitOps-owned resources: commit the overlay change and let the controller reconcile; avoid racing it with imperative `kubectl apply`. Always evaluate prune and rollback consequences.

## Acceptance

Deliver focused base/overlay changes, environment renders or non-sensitive diffs, validation evidence, identity and deletion risks, and explicit untested assumptions. Never claim cluster acceptance from a local build.

## Official docs

- https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
- https://kubectl.docs.kubernetes.io/references/kustomize/
- https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/
