---
name: kubernetes-security
description: Audit, design, or enforce Kubernetes application and namespace security. Use for Pod Security Standards/Admission, securityContext, RBAC/ServiceAccounts, Secrets, NetworkPolicy, image signature/admission rules, user namespaces (hostUsers false), runtime isolation with Kata/gVisor, and volume permissions.
---

# Kubernetes workload security

Apply layered, **verifiable least privilege**. Identify cluster and runtime capabilities before introducing restrictions; exception paths must be narrow, owned and time-limited. Do not assume namespaces alone isolate untrusted tenants.

## Workflow

1. Inspect Kubernetes version, Linux/Windows nodes, runtime/CRI, CNI policies, CSI drivers, Pod Security Admission labels, admission policies, ServiceAccounts/RBAC, workload identity, secrets, storage and observability needs.
2. Threat-model identities, image trust, network peers, host resources and data paths. Select a baseline (`restricted` for ordinary Linux apps) and document justified exceptions for system agents or specialized workloads.
3. Harden namespace, Pod, container, network and supply chain; test storage and networking under realistic UID/security profiles, including startup and debug paths.
4. Render/validate policies and simulate allowed + denied operations. Report which rules the cluster actually enforces; don't apply irreversible production policy changes without authorization.

## Pod and namespace baseline

- Enforce version-pinned Pod Security Admission `restricted` for normal app namespaces; use `enforce`, `audit`, `warn` as appropriate and introduce enforcement after observing violations. Privileged infrastructure workloads belong in separately reviewed namespaces. Check **effective version** and rollout compatibility.
- Use trusted images pinned to digests; verify signer/issuer/source and allowed registries through suitable admission (Kyverno, Gatekeeper integrations or image policy) **only if configured**. SBOM, signature and provenance are different artifacts.
- Disable default API token mounts (`automountServiceAccountToken: false`) when not needed; otherwise use dedicated ServiceAccount with namespaced, minimal RBAC verbs/resources and projected short-lived tokens. Workload identity is preferable to static cloud credentials.
- Prefer non-root (numeric UID or platform-assigned arbitrary UID); use `readOnlyRootFilesystem`, explicit writable tmp/data mounts, `allowPrivilegeEscalation: false`, `seccompProfile: RuntimeDefault`, `capabilities.drop: [ALL]`, and required AppArmor/SELinux policy when supported. `readOnlyRootFilesystem` is **not** required by Restricted PSA.
- No `privileged`, broad `hostPath`, container runtime sockets, host PID/IPC/network namespaces or unnecessary capabilities/devices. Default-deny by NetworkPolicy where CNI enforces it, but add reviewed DNS and necessary application traffic first.

### Container/Pod defaults (Linux)

```yaml
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    seccompProfile: {type: RuntimeDefault}
  containers:
    - name: app
      image: registry.example.com/team/app@sha256:REPLACE_WITH_REAL_DIGEST
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities: {drop: [ALL]}
```

This is a **security fragment, not a deployable Pod**. Validate runtime user, available writable paths and storage rights. Add back a capability only after demonstrating necessity; `NET_BIND_SERVICE` may be permitted under Restricted.

## Secrets and data access

- Prefer external secret manager + workload identity where available; enable encryption at rest, least-privilege RBAC and rotation. Base64 Kubernetes Secret values are **not encryption**. Avoid env/CLI args for credentials when file mounts are feasible.
- A principal able to create Pods in a namespace can potentially read Secrets by mounting them; RBAC restrictions on `get secrets` alone are insufficient. Avoid sharing security boundaries across trust levels.
- Use per-container secret mounts, mode/UID compatible with the process; audit access and avoid secret values in logs, manifests, Helm values and CI artifacts. Mounted secret changes are eventual and aren't automatically reread by apps.
- `fsGroup` is not universally required: check image UID/GID, CSI behavior, `fsGroupChangePolicy`, mount costs and access modes. Don't recursively `chown` shared PVCs or modify hostPath ownership without explicit approval.

## Network isolation

- NetworkPolicy is **additive**, enforced by a supporting CNI, and operates at L3/L4; it is not HTTP route authorization or a general host firewall. Egress and ingress must both allow a flow when both directions are isolated.
- Start with verified DNS egress and required upstreams; include necessary ingress from Services/Gateways and monitoring. Test **allowed and denied** cases; account for node/hostNetwork/CNI-specific behavior. A NetworkPolicy only restricts peers if policy selects the right Pods and CNI implements it; assert denial from a forbidden Pod and success from an allowed Pod in a nonproduction namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-app
spec:
  podSelector:
    matchLabels: {app: app}
  policyTypes: [Ingress, Egress]
```

The policy denies **both directions** for selected Pods until other allowing policies exist; do **not** apply blindly to a live application.

## User namespaces and stronger isolation

- `spec.hostUsers: false` maps Pod UIDs/GIDs to unprivileged host IDs; it is **not** the same as `runAsNonRoot` and does not remove the need for seccomp/capability/LSM/network controls. Kubernetes v1.36+ marks user namespaces stable, but kernel, CRI, nodes and storage must still support them.
- `runAsUser`, `runAsGroup`, and `fsGroup` are IDs **inside** the Pod. Test idmapped volume support; NFS and some CSI/filesystem combinations are incompatible. `hostNetwork`, `hostPID`, `hostIPC` and raw-block `volumeDevices` are unavailable with user namespaces. PSA non-root checks differ for user-namespaced Pods; confirm actual cluster version.
- Choose gVisor/Kata via supported RuntimeClass only after verifying CNI, CSI, syscalls, architecture, metrics/logging, startup latency and overhead. Kata VM isolation is stronger against host-kernel attacks but host eBPF may not observe guest process activity; gVisor also changes syscall observability. Consider in-guest instrumentation/sidecars, and treat sidecars as shared trust boundary.
- Never equate `hostUsers: false`, Kata, gVisor and `runAsNonRoot`; they protect different boundaries.

## Validation

```sh
kubectl get ns "$NS" --show-labels
kubectl auth can-i get secrets -n "$NS" --as "system:serviceaccount:$NS:$SA"
kubectl apply --dry-run=server -n "$NS" -f rendered.yaml
kubectl describe pod -n "$NS" "$POD"
```

Use a negative admission fixture: Pod requesting `privileged: true`, hostPath or forbidden capability must fail when the corresponding policy is expected. Test ServiceAccount RBAC with impersonation and, where permitted, a real Pod/Secret mount scenario (namespace Pod creators may read Secrets through mounts). Check admission errors, actual UID mapping when permitted (`cat /proc/self/uid_map` inside a suitable container), PVC permission tests, DNS/egress and blocked host access. Server-side dry-run validates only reachable admission behavior, **not** effective runtime/network isolation; test the latter separately. Avoid shell exec assumptions in distroless images.

## Deliverable

Return least-privilege configuration/policy diff, threat assumptions, prerequisites (CNI/CSI/CRI/RuntimeClass/PSA), expected allowed/denied behaviors, tests and precisely justified exceptions. Do not claim a control works without enforcement evidence.

## Official references

- https://kubernetes.io/docs/concepts/security/application-security-checklist/
- https://kubernetes.io/docs/concepts/security/pod-security-standards/
- https://kubernetes.io/docs/concepts/security/secrets-good-practices/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/
