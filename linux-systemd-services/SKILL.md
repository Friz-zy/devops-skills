---
name: linux-systemd-services
description: Write, review, harden, test and troubleshoot systemd service, socket, timer and mount units on Linux. Use for unit dependencies, service identities, credentials, filesystem sandboxing, resource limits, restart policies, journald, service activation, upgrade and rollback. Use ansible-automation to distribute units across hosts.
---

# systemd services and hardening

Use the **least privilege compatible with the service contract**; systemd isolation options can break DNS, networking, shared libraries, hardware access or persistent state. Never enable/restart production units without approval.

## Workflow

1. Inspect `systemd --version`, distro, user/system manager, app binary and exec paths, required network/IPC/files, credentials, UID, sockets, restart semantics, state ownership, dependencies and watchdog support.
2. Define service type, readiness/start/stop behavior, failure modes and writable directories. Prefer package-owned unit + drop-in over replacing vendor units.
3. Add compatible hardening one group at a time; validate unit and test from clean start, reboot, shutdown, crash and upgrade in a safe environment.

## Rules

- `ExecStart=` must be absolute and predictable; avoid `sh -c`/shell unless unavoidable. Choose `Type=simple`/`exec`/`notify` according to the actual daemon: `Type=notify` requires service notifications; `WatchdogSec` requires periodic watchdog notifications.
- Run dedicated non-root `User=`/`Group=` or `DynamicUser=yes` when persistence/ACL semantics permit. With `DynamicUser`, use `StateDirectory=`, `CacheDirectory=`, `LogsDirectory=` rather than hardcoded home ownership; do not assume stable UID across restarts.
- Use `LoadCredential=`/`LoadCredentialEncrypted=` or secure external secrets delivery for runtime secrets when supported; restrict file permissions, avoid environment variables/command args and `systemctl show`/journal exposure. Confirm credential lifecycle and restart behavior.
- Start with `NoNewPrivileges=yes`, `PrivateTmp=yes`, `ProtectSystem=strict`, `ProtectHome=yes` **only after** listing required writable/visible paths; `StateDirectory=`/`RuntimeDirectory=` or narrow `ReadWritePaths=` can permit writes. Add `PrivateDevices`, `ProtectKernelTunables`, `ProtectControlGroups`, `RestrictSUIDSGID`, `LockPersonality`, `CapabilityBoundingSet` and syscall/AF restrictions as compatible; don't blindly block networking, JIT, namespaces, `/proc` or device access.
- Grant only required Linux capabilities (`AmbientCapabilities`/`CapabilityBoundingSet`) and validate no untrusted environment controls executable loading. Sandbox knobs are defense in depth, not a replacement for patching, SELinux/AppArmor or kernel hardening.
- Bound CPU/memory/tasks via cgroups (`MemoryMax`, `CPUQuota`, `TasksMax`) based on measured needs; understand `OOMPolicy`. Use `Restart=on-failure` and bounded `RestartSec`/`StartLimitIntervalSec`/`StartLimitBurst` to avoid loops. Do not auto-restart deliberately completed one-shot units.
- Use `After=` for order **only** and `Wants=`/`Requires=` for dependency semantics. `network-online.target` is not a universal readiness guarantee; app-level reconnect/retry remains necessary. For timers, check `Persistent=`, time zone, missed-run behavior and duplicate/exclusive execution.
- Log to journald/stdout; set retention/rotation at platform level; redact secrets. Add meaningful `ExecStartPre`/`ExecCondition` only if side effects and permissions are understood. `ExecReload` must perform real reload, not a blind restart.
- For safe upgrades, stage unit and binary atomically where feasible, `daemon-reload`, restart in approved window, verify health, and preserve the previous release. Test SIGTERM handling and bounded `TimeoutStopSec`.

## Example (adapt to app contract)

```ini
[Unit]
Description=Example API
After=network.target

[Service]
Type=exec
User=appsvc
Group=appsvc
ExecStart=/usr/local/bin/example-api
StateDirectory=example-api
RuntimeDirectory=example-api
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
CapabilityBoundingSet=
AmbientCapabilities=
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30s

[Install]
WantedBy=multi-user.target
```

Requires pre-created `appsvc` account, executable and correctly configured write paths; `StateDirectory` is not automatically an application config path. Blank capability sets suit only applications that need none.

## Validation (safe first)

```sh
systemd-analyze verify ./example-api.service
systemd-analyze security example-api.service   # score is heuristic, not proof
systemctl cat example-api.service
systemctl show example-api.service -p User -p DynamicUser -p ProtectSystem
journalctl -u example-api.service -n 100 --no-pager
# Authorized only: systemctl daemon-reload; systemctl restart example-api.service
```

Test read-only filesystem, allowed/denied file paths, socket/network binding, credential access, failure backoff, stop/restart and shutdown timeout. Note `systemd-analyze verify` does not replace a live service test; user services and older systemd releases support a different subset of sandbox options.

## Handoff

Provide unit/drop-in, exact version compatibility, required directories/capabilities, executable/identity assumptions, validation evidence, rollout/rollback and known isolation exceptions.

## Sources

- https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html
- https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
- https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html
- https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html
