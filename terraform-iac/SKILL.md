---
name: terraform-iac
description: Design, review, refactor, test and safely change infrastructure with Terraform or OpenTofu. Use for HCL modules, provider constraints, remote state, imports/moves, plans, resource lifecycle, policy checks and infrastructure CI.
---

# Terraform / OpenTofu changes

Treat state and the reviewed plan as production-critical artifacts. Prefer the smallest declarative change; never run `apply`, `destroy`, `import`, `state`, `force-unlock` or backend migration against real infrastructure without explicit approval.

## Workflow

1. Identify Terraform/OpenTofu and provider versions, target account/project, workspace, backend, state ownership, deployment identity and existing resource addresses. Verify the directory **and credentials** point to the intended environment.
2. Model desired resources with stable addresses; review dependencies, costs, permissions, data migration, replacement/destroy risk, and rollback feasibility. Don't invent provider attributes or resource IDs.
3. Format, validate, scan, test and produce a reviewed plan using authorized **read/plan-only** credentials; report planned creates/updates/replacements/deletes and any unknown values.
4. After approval, apply the **same saved plan** with protected credentials and state lock; verify actual outcomes and drift. Saved plans may embed secrets and may become stale: secure them and abort/replan on conflicting state changes.

## Module and state rules

- Pin `required_version`, provider versions in `required_providers`, and trusted module sources to reviewed versions/commits. Commit `.terraform.lock.hcl`; don't commit `.terraform/`, `*.tfstate*`, `*.tfplan` or plaintext `.tfvars` secrets. Update locks intentionally and review cross-platform checksums.
- Separate independent environments/trust domains into separate root modules, backends and access boundaries; Terraform CLI workspaces alone are not an isolation/security boundary. Keep module interfaces small; use typed variables, documented outputs and no secrets in default values.
- Use a remote backend with documented **locking, encryption, versioning/backup and least-privilege access**. Confirm backend actually supports locking; never use `-lock=false` as a workaround. Rehearse recovery/restore; avoid raw state edits or casual `state push`.
- `sensitive = true` redacts display but **does not** keep a value out of state/plan. Prefer short-lived workload identity and provider-supported `ephemeral`/write-only arguments when supported; audit all state, plan, logs and output access.
- Use stable `for_each` keys rather than index-sensitive `count` for named resources; avoid renaming addresses accidentally. Use `moved` blocks for refactors, config-driven `import` blocks for adopted resources, and `removed` blocks only with explicit ownership decisions. Verify provider/API side effects.
- Use `lifecycle.prevent_destroy` narrowly for irreplaceable resources; it is not absolute protection (configuration can be removed). `ignore_changes` can hide drift; justify each field. Avoid `null_resource`/provisioners for routine config management.
- Bound risk with input `validation`, resource `precondition`/`postcondition`, policy-as-code (OPA/Conftest, Sentinel, etc.) and cost/security checks. `check` blocks may warn without blocking: don't mistake them for hard gates.
- Don't rely on `-target` for regular deployments; don't bypass review on a resource replacement, `create_before_destroy`, renames, bucket/database deletion or CIDR exposure. Verify uniqueness, quotas and cloud eventual consistency.

## Useful checks

```sh
terraform version                                # or: tofu version
terraform fmt -check -recursive
terraform init -backend=false -lockfile=readonly # static module validation; not a plan
terraform validate
terraform test                                  # inspect tests: may provision real resources
# Optional, if installed: tflint; checkov; trivy config .
```

For a reviewed plan **only in an approved, correctly configured environment**:

```sh
terraform init -lockfile=readonly
terraform plan -lock-timeout=5m -out=change.tfplan
terraform show -no-color change.tfplan           # may reveal sensitive values
terraform show -json change.tfplan > change.json # protect as a secret artifact
```

- Do not run plan on untrusted PR HCL with privileged cloud credentials; data sources/providers can access cloud APIs during plan. Isolate runners and use narrowly scoped read permissions.
- In CI, `terraform plan -detailed-exitcode` means `0` no changes, `2` changes, `1` error; handle deliberately. Inspect `resource_changes[*].change.actions` and require approval for delete/replace.
- Test refactors against a **copy** of relevant state in isolation, and verify no unexpected delete/create. For live changes, recheck state lock, identity and digest of approved plan before applying.
- Test rollback by reverting config or restoring service/data appropriately; `terraform destroy` is not rollback. Drift checks, state recovery and destructive-change drills belong to the operational runbook.

## Output

Provide focused HCL/diff, version/backend assumptions, plan impact (especially destroys), validation results, access boundaries, tests, approval requirements and recovery plan. Never claim apply success without evidence.

## Sources

- https://developer.hashicorp.com/terraform/language/state/locking
- https://developer.hashicorp.com/terraform/language/manage-sensitive-data
- https://developer.hashicorp.com/terraform/language/validate
- https://developer.hashicorp.com/terraform/language/tests
- https://opentofu.org/docs/language/state/encryption/
