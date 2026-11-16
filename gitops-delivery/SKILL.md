---
name: gitops-delivery
description: Design, audit, migrate and troubleshoot GitOps delivery with Argo CD or Flux. Use for Git source boundaries, Application/AppProject/ApplicationSet, Flux Kustomization/HelmRelease, RBAC, decryption, sync, drift, pruning, dependency ordering, promotion, reconciliation and rollback.
---

# GitOps delivery

Git defines intended state; the controller performs reconciliation. Protect the **source → renderer → controller identity → cluster** trust chain. Never enable automatic prune/self-heal or change production sources without approval.

## Workflow

1. Inspect GitOps product/version, watched repo/revision/path, cluster/environment ownership, project/tenant RBAC, deployed CRDs, image artifacts, secrets, admission policies, health and rollback path.
2. Identify each resource's **single controller/owner**; detect overlaps with Helm releases, Terraform, manual `kubectl`, operators or another GitOps application.
3. Stage the smallest Git change; render and validate every affected environment. Preview **deletions, renames, immutable selector changes and drift** before enabling sync/prune.
4. After approved reconciliation, verify revision, health, image digest, access controls and expected drift behavior. Rollback by reverting **Git desired state**; verify data/schema compatibility.

## Security and ownership boundaries

- Pin source references (Git commit/reviewed tag or signed protected branch), Helm charts/dependencies and image **digests**. Restrict write access to Git source and CI release paths; protect CODEOWNERS/approvals. No plaintext secrets in Git, generated manifests or controller logs; use SOPS with managed keys or External Secrets and test decryption RBAC.
- Give each tenant a separate, least-privilege controller identity and destination scope. Limit accessible Git repositories, namespaces, clusters, resource kinds, impersonation and cross-namespace references; do not assume a namespace alone isolates tenant-owned cluster-scoped resources.
- **Argo CD:** restrict `AppProject.spec.sourceRepos`, `destinations`, cluster/namespace resource allowlists/denylists, project roles and permitted Application source namespaces. Review ApplicationSet generators/templates: generator inputs must not grant arbitrary source/destination access. Do not broadly grant repo credentials or `default` project access.
- **Flux:** use `spec.serviceAccountName` for tenant Kustomizations/HelmReleases, restrict cross-namespace references and remote bases, limit sources/controller permissions; prefer scoped, verified source artifacts. Keep controller version and CRDs compatible.
- `prune`, `allowEmpty`, `force`, `replace`, resource renaming and orphan deletion can destroy PVCs/CRDs/data. Explicitly review these flags, retention policies and finalizers; never treat `prune: true` as safe by default.
- Do not mask genuine drift with blanket `ignoreDifferences`. Target exact controller-mutated fields and verify the difference is expected; check whether ignored fields are still applied depending on sync options.

## Release and dependency rules

- CI builds/scans/signs immutable artifacts; promotion updates a reviewed digest in the environment Git source. Prefer one source of truth for the image and values per environment; avoid cross-environment mutable references.
- For Argo CD, use sync waves/hooks only for real ordering constraints; check health and CRD availability before custom resources. Manual `argocd app sync` on automatically managed resources needs owner approval.
- For Flux, `dependsOn`, `healthChecks`, `wait`/timeouts and HelmRelease remediation must match actual readiness; don't create dependency cycles or suppress failures. Check whether CRDs are installed before CRs.
- Automated sync must not be confused with rollback guarantees. Argo auto-sync, self-heal and prune can override manual fixes; ApplicationSet-managed Applications require changing their generator/source configuration.
- Disallow unreviewed production image auto-updates; promotion must account for signed artifacts, backward-compatible DB migrations and traffic/progressive deployment policies.

## Validation

```sh
# Select tools actually used by the repository:
kubectl kustomize ./overlays/staging > rendered.yaml
# OR: helm template app ./chart -f values/staging.yaml > rendered.yaml
# Optional: kubeconform -strict -summary rendered.yaml
# Argo read-only: argocd app get APP; argocd app diff APP
# Flux read-only: flux get sources all -A; flux get kustomizations -A
# Flux: flux build kustomization NAME --namespace NS  (check installed version/flags)
```

- Confirm no unexpected inventory removal or resource takeover; compare Git desired objects with current ownership labels/annotations and cluster RBAC. `kubectl --dry-run=server` checks admission, **not** controller reconciliation or Secret decryption.
- Test both allow and deny: tenant cannot target another cluster/namespace, use unapproved repo/CRD, access another tenant's Secret, or deploy unsigned/unapproved images (when admission policy enforces this).
- Test sync on a nonproduction environment including missing CRD, invalid Secret/decryption, failed health, partial sync and restore of expected resources. Verify observability of OutOfSync/NotReady and stalled reconciliation.

## Handoff

Show exact Git path/ref and destination, ownership/RBAC, pruning risk, environment render/policy evidence, promotion/rollback procedure and unresolved controller compatibility.

## Sources

- https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
- https://argo-cd.readthedocs.io/en/stable/user-guide/projects/
- https://fluxcd.io/flux/security/best-practices/
- https://fluxcd.io/flux/components/kustomize/kustomizations/
