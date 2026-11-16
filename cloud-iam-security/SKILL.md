---
name: cloud-iam-security
description: Design, review and test least-privilege cloud IAM policies and workload identities, primarily AWS IAM/STS, EKS Pod Identity/IRSA and CI OIDC, with equivalent Azure/GCP controls. Use for trust policies, role assumption, permissions boundaries, ABAC, cross-account access, access analyzer, credential lifecycle and negative authorization tests.
---

# Cloud IAM and workload identity

Define **who can assume which identity, for which action, on which resources, under which conditions**. Permissions and trust are separate boundaries. Never broaden live roles or change production trust policies without approval.

## Workflow

1. Identify provider/account/tenant, trust boundary, caller (human/CI/workload), target APIs/resources, current grants, org-level guardrails, logging and workload topology. Verify actual account identity before testing.
2. Write the minimum action/resource/condition matrix and corresponding trust policy; inspect transitive calls (`iam:PassRole`, KMS, STS, S3 ACL/bucket policies, etc.) and cross-account/resource-policy effects.
3. Validate with policy syntax/analyzers, simulate expected allow/deny cases, then test with isolated short-lived credentials. Record policy ownership and rollback path.

## Rules

- Prefer human federation/SSO + MFA and workload **short-lived tokens/roles** over access keys. Limit IAM user keys to unavoidable legacy cases with rotation, detection and expiry. No credentials in code, image, Terraform state, environment dumps, logs or CI artifacts.
- Separate role **trust** (assume conditions) from permission policy. Name exact principals; restrict federated OIDC by `aud`, subject, repository/branch/environment/workflow as supported. Validate case-sensitive claim semantics and token issuer; test that forks/unprotected branches cannot assume production roles.
- AWS: prefer `Resource` ARN restrictions, service-specific condition keys, `aws:PrincipalOrgID` and source constraints **when supported by that API**. Be careful with `NotAction`, wildcards and `iam:PassRole`; enforce permitted role scope and `iam:PassedToService` where suitable.
- Evaluate explicit deny, SCP/RCP, permissions boundaries, identity/session/resource policies and VPC endpoint policies. A boundary is a maximum, **not** a grant. Simulator results alone may not model every cross-account/resource/session context.
- EKS: use dedicated service account identity via Pod Identity or IRSA according to cluster SDK/agent/STS compatibility. Restrict node instance metadata (IMDS); don't assume Pod Identity automatically isolates credentials when host networking or host privileges are available.
- For ABAC, control who may set/change **tags** as well as who may use tagged resources. Restrict policy/role self-modification and privilege escalation, resource policy edits, secret reads, KMS grants, cross-account assume-role chains.
- Avoid `Action: '*'`, `Resource: '*'`, `AdministratorAccess` and blanket trust. Where a service requires wildcards, constrain with supported conditions and document explicit exception, owner and expiration. Do not add arbitrary `Deny` clauses without understanding affected service integrations.
- Restrict audit-log access and retain CloudTrail/equivalent; monitor unused permissions, risky grants and suspicious role assumption. Rotate/revoke roles and update clients without outages when possible.

## AWS checks (where installed and authorized)

```sh
aws sts get-caller-identity
aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type IDENTITY_POLICY
# Test policy simulator for actions/resources/contexts when supported:
# aws iam simulate-principal-policy --policy-source-arn ROLE_ARN --action-names ACTION --resource-arns ARN
```

- Test **allowed** service call and **denied** cross-tenant, wrong repo/ref/audience, wrong namespace/SA, wrong resource, privilege-escalation and unauthorized tag modification. Policies should fail closed when claims are absent.
- Use read-only audit credentials first; policy simulation has limitations for resource-based/cross-account context and must be followed by scoped integration tests. Never publish raw `get-role`/credential outputs containing sensitive account identifiers into public logs.
- Validate OIDC rotation/JWKS, session expiration, refresh and token audience against the actual CI/cloud SDK. Require independent review on grants that affect production data or role delegation.

## Handoff

Provide action-resource-principal matrix, identity/trust policies, required organizational guardrails, positive/negative results, exceptions, rollout owner and revocation/rollback steps.

## Sources

- https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-validation.html
- https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html
- https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
