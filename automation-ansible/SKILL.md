---
name: ansible-automation
description: Write, review, refactor, and validate Ansible playbooks, roles, inventories, handlers, templates, variables, and automation pipelines. Use for repeatable server configuration, package/service management, rolling updates, privilege escalation, Ansible Vault, collection dependencies, ansible-lint, check mode and idempotency testing.
---

# Ansible automation

Make infrastructure changes idempotent, reviewable, least-privileged and safe on repeated runs. Prioritize purpose-built modules over shell commands. Never change production hosts without authorization.

## Workflow

1. Inspect `ansible-core` version, target OS/facts, inventory and group membership, `ansible.cfg`, existing roles/collections, connection/become method, secrets and maintenance/rollback constraints.
2. Identify the **desired end state** and smallest host group. Prefer existing patterns and modules; choose roles only for reusable or sufficiently complex work.
3. Validate inventory, dependencies, syntax and lint; check on a safe host; then perform an authorized staged run and a second run to verify idempotency.

## Rules

- Name every play, task and handler descriptively. Use module FQCNs (`ansible.builtin.*`, `community.*`) and canonical module names. Keep tasks small and defaults explicit (`state`, ownership, mode, enabled).
- Prefer `package`/`apt`/`dnf`, `user`, `file`, `copy`, `template`, `service`/`systemd_service`, `get_url`/`unarchive` etc. Use `command` with `argv` instead of `shell` when shell expansion/pipes are unnecessary. For shell, quote untrusted data, set safe executable/pipeline behavior and document why no module fits.
- Preserve idempotency: templates/files should converge; use `notify` + handlers for restarts, not unconditional restarts. For commands that mutate, use supported `creates`/`removes`, status checks or truthful `changed_when`/`failed_when`; `changed_when: false` must not hide real changes.
- For reusable roles, declare inputs in `meta/argument_specs.yml` with required fields and types; make `ansible.builtin.assert` checks only for cross-field invariants or older versions without role specs. Test that invalid arguments fail before modifying hosts.
- Keep inventory, group/host vars and role defaults predictable. Role `defaults/main.yml` is for overridable defaults; avoid putting user-tunable settings into higher-precedence role vars. Understand variable precedence and avoid ad hoc `-e` overrides in production.
- Prefer `roles/<name>/{tasks,handlers,templates,defaults,vars}/` for reused logic, with `site.yml` as the entry point. Use static `import_*` when the task graph is fixed; dynamic `include_*` when runtime selection is required. Avoid unnecessary nesting.
- Pin `ansible-core` and third-party roles/collections for reproducibility; keep `requirements.yml` in version control. Verify collection/module compatibility on both control and target nodes; don't assume external collections ship with `ansible-core`.
- Scope `become: true` only to tasks/plays that need it; do not embed `sudo` in shell. Keep SSH host key checking enabled and use trusted hosts/keys. Avoid arbitrary `shell` with `become` and untrusted variables.
- Store secrets with Ansible Vault or an approved external secrets backend; never commit plaintext credentials. `no_log: true` on sensitive tasks; `diff: false` on secret-bearing templates. Vault does **not** prevent runtime/log/process leaks. Don't print secrets via `debug`, facts or registered results. Protect CI log/artifact permissions.
- Use safe file modes as **quoted strings** (`'0640'`), explicit owner/group; write complete configs with `template` rather than repeatedly editing individual lines. Use module `validate` for configuration syntax when supported before replacing a live config.
- For rolling fleet changes, use `serial`, safe batch size and health checks. Avoid broad `hosts: all` for destructive tasks. `run_once` executes per `serial` batch; delegated tasks and handlers require care. Use `block`/`rescue` for cleanup and explicit recovery but do not claim atomic transactions.
- Check mode is **simulation**, not proof: tasks with unsupported check mode may skip, and registered-variable conditionals may differ. `--diff` can expose secrets; use only when safe. Test actual operation and verify a repeated run produces no unintended changes.

## Minimal pattern

```yaml
---
- name: Configure application service
  hosts: app_servers
  become: true
  serial: 1
  tasks:
    - name: Install application package
      ansible.builtin.package:
        name: myapp
        state: present

    - name: Install application configuration
      ansible.builtin.template:
        src: templates/myapp.yml.j2
        dest: /etc/myapp/config.yml
        owner: root
        group: root
        mode: '0644'
      notify: Restart myapp

    - name: Ensure application service is enabled and running
      ansible.builtin.service:
        name: myapp
        enabled: true
        state: started

  handlers:
    - name: Restart myapp
      ansible.builtin.service:
        name: myapp
        state: restarted
```

Assumes the package, service, template path and target OS exist; adapt file permissions to app needs. Add a real health check before moving to the next batch. For sensitive configs, use `mode: '0600'`, `no_log: true`, `diff: false` and ensure the service can read them.

## Validation and rollout

```sh
ansible --version
ansible-galaxy collection install -r requirements.yml  # only if present
ansible-inventory -i inventories/staging/hosts.yml --graph
ansible-playbook -i inventories/staging/hosts.yml site.yml --syntax-check
ansible-lint --strict  # if installed
ansible-playbook -i inventories/staging/hosts.yml site.yml --limit canary --check
# After authorized apply on canary, run again and verify no unexpected 'changed'.
```

- Use `--check --diff` only for non-sensitive work; redact CI output and never use `--diff` indiscriminately on encrypted settings that render to plaintext files.
- For critical changes, test on one host, then small batches with health verification, then the fleet. Record backups/migration/reboot steps and clear recovery action. Use `--limit` to constrain blast radius; check inventory membership before production runs.
- Use Molecule (if maintained/configured) or disposable VMs/containers to verify apply → rerun produces zero unintended changes; assert config semantics and service health, not merely task status. Include negative tests for invalid variables, bad downloads, missing privilege, nonexistent group and unreachable host; never test against the production inventory by default.
- `ansible-lint` is a quality gate, not a security guarantee; review shell privilege, download verification, token exposure, callback plugins, inventory sourcing and changed/failed semantics manually.

## Acceptance

Return complete changed playbook/role files, expected inventory and versions, secrets handling, validation evidence, idempotency results, rollout scope and known recovery limits. Never claim a dry-run guaranteed safety or report hosts changed without execution evidence.

## Official docs

- https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_best_practices.html
- https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_checkmode.html
- https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_handlers.html
- https://docs.ansible.com/projects/lint/
