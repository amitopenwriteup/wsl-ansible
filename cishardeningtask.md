# Ubuntu CIS-Style Hardening Playbook — Incremental Build (Task-by-Task)

**Companion to:** *Workshop: Compliance-as-Code & Security Hardening for Ubuntu (via Ansible)*
**Purpose:** Build the check → remediate → verify playbook one task at a time, in the exact order the rows appear in §4.1 (Audit/Check), §4.2 (Remediation), and §4.3 (Verify), so each addition can be reviewed and validated against the current hardening automation before the next is appended.

---

## How to use this document

- Each task below corresponds to exactly one row of the module table in the referenced section.
- Tasks are appended in row order — do not skip ahead.
- A small, consistent set of sample controls (SSH root login, `/tmp` permissions, disallowed package, unwanted service, sysctl parameter, account lockout, firewall, auditd rule) is threaded through the rows so the build stays concrete rather than abstract.
- After each task, re-run against the non-prod test group with `--check` (dry-run) first, per §3.2's discussion point on audit-only mode, before applying for real.
- The final section (§5) shows the fully assembled playbook after all rows from §4.1–§4.3 have been appended.

---

## §4.1 Audit / Check (read-only) — task-by-task

### Task 1 of 7 — Read a config file's current value

| Row | Module |
|---|---|
| Read a config file's current value | `ansible.builtin.slurp` or `ansible.builtin.lineinfile` (check mode) |

```yaml
---
- name: Ubuntu CIS-Style Hardening Playbook (sample controls)
  hosts: hardening_targets
  become: true
  vars:
    compliance_exceptions: []
    compliance_results: []

  tasks:
    - name: "CIS-5.2.10 - Check current PermitRootLogin line"
      ansible.builtin.slurp:
        src: /etc/ssh/sshd_config
      register: sshd_config_raw
      when: "'CIS-5.2.10' not in compliance_exceptions"
      tags: [audit, "CIS-5.2.10"]
```

> Read-only — `slurp` pulls the file content back base64-encoded for inspection/Jinja parsing without touching the host. This is deliberately non-destructive per §3's definition of "audit."

---

### Task 2 of 7 — Check file permissions/ownership

| Row | Module |
|---|---|
| Check file permissions/ownership | `ansible.builtin.stat` |

```yaml
    - name: "CIS-1.1.x - Check /tmp permissions"
      ansible.builtin.stat:
        path: /tmp
      register: tmp_perms
      when: "'CIS-1.1.x' not in compliance_exceptions"
      tags: [audit, "CIS-1.1.x"]
```

---

### Task 3 of 7 — Check installed/forbidden packages

| Row | Module |
|---|---|
| Check installed/forbidden packages | `ansible.builtin.package_facts` |

```yaml
    - name: "CIS-2.x - Gather package facts"
      ansible.builtin.package_facts:
      tags: [audit, "CIS-2.x"]

    - name: "CIS-2.x - Check for disallowed telnet package"
      ansible.builtin.set_fact:
        telnet_present: "{{ 'telnet' in ansible_facts.packages }}"
      when: "'CIS-2.x' not in compliance_exceptions"
      tags: [audit, "CIS-2.x"]
```

---

### Task 4 of 7 — Check running/enabled services

| Row | Module |
|---|---|
| Check running/enabled services | `ansible.builtin.service_facts` |

```yaml
    - name: "CIS-2.y - Gather service facts"
      ansible.builtin.service_facts:
      tags: [audit, "CIS-2.y"]

    - name: "CIS-2.y - Check for unwanted avahi-daemon service"
      ansible.builtin.set_fact:
        avahi_enabled: "{{ ansible_facts.services['avahi-daemon.service'].status | default('not-found') == 'enabled' }}"
      when: "'CIS-2.y' not in compliance_exceptions"
      tags: [audit, "CIS-2.y"]
```

---

### Task 5 of 7 — Check kernel/sysctl parameters

| Row | Module |
|---|---|
| Check kernel/sysctl parameters | `ansible.builtin.command` (`sysctl -n <param>`) with `changed_when: false` |

```yaml
    - name: "CIS-3.x - Check current ip_forward value"
      ansible.builtin.command: sysctl -n net.ipv4.ip_forward
      register: ip_forward_check
      changed_when: false
      when: "'CIS-3.x' not in compliance_exceptions"
      tags: [audit, "CIS-3.x"]
```

> No pure declarative built-in exists for reading sysctl, per the table's note — `command` is used deliberately here and marked `changed_when: false` so it never reports a false change.

---

### Task 6 of 7 — Check user/group account settings

| Row | Module |
|---|---|
| Check user/group account settings | `ansible.builtin.getent` |

```yaml
    - name: "CIS-4.x - Look up account entry via NSS"
      ansible.builtin.getent:
        database: passwd
        key: "{{ unused_account_name | default('games') }}"
      when: "'CIS-4.x' not in compliance_exceptions"
      tags: [audit, "CIS-4.x"]
```

> Read-only NSS lookup — no shell parsing of `/etc/passwd` needed.

---

### Task 7 of 7 — Validate a value against expected compliance state

| Row | Module |
|---|---|
| Validate a value against expected compliance state | `ansible.builtin.assert` |

```yaml
    - name: "CIS-3.x - Assert ip_forward is disabled"
      ansible.builtin.assert:
        that: ip_forward_check.stdout | trim == '0'
        fail_msg: "CIS-3.x non-compliant: net.ipv4.ip_forward is enabled"
        success_msg: "CIS-3.x compliant: net.ipv4.ip_forward is disabled"
      when: "'CIS-3.x' not in compliance_exceptions"
      register: cis_3x_audit_result
      ignore_errors: true
      tags: [audit, "CIS-3.x"]
```

> This is the central audit pattern per the table: `register` the raw check, then `assert` with a `fail_msg` naming the control ID. `ignore_errors: true` here so a failed assert records non-compliance rather than halting the whole audit run — the actual stop/flag decision belongs to §4.3's `fail` task, not here.

**§4.1 complete.** Validate with `--tags audit --check` against the test group before moving to §4.2.

---

## §4.2 Remediation — task-by-task

### Task 1 of 11 — Enforce a config line/value

| Row | Module |
|---|---|
| Enforce a config line/value (e.g., `sshd_config`) | `ansible.builtin.lineinfile` |

```yaml
    - name: "CIS-5.2.10 - Remediate PermitRootLogin"
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart sshd
      when:
        - "'CIS-5.2.10' not in compliance_exceptions"
        - "'permitrootlogin yes' in (sshd_config_raw.content | b64decode | lower)"
      tags: [remediate, "CIS-5.2.10"]
```

---

### Task 2 of 11 — Enforce a config block

| Row | Module |
|---|---|
| Enforce a config block | `ansible.builtin.blockinfile` |

```yaml
    - name: "CIS-5.x - Enforce SSH hardening block"
      ansible.builtin.blockinfile:
        path: /etc/ssh/sshd_config.d/99-cis-hardening.conf
        create: true
        mode: '0644'
        block: |
          Protocol 2
          X11Forwarding no
          MaxAuthTries 4
      notify: restart sshd
      when: "'CIS-5.x' not in compliance_exceptions"
      tags: [remediate, "CIS-5.x"]
```

> `blockinfile` is preferred here over repeated `lineinfile` calls because these settings are a clearly-delimited multi-line group, per the table's note.

---

### Task 3 of 11 — Full template-managed config file

| Row | Module |
|---|---|
| Full template-managed config file | `ansible.builtin.template` |

```yaml
    - name: "CIS-4.y - Deploy fully-managed login.defs"
      ansible.builtin.template:
        src: templates/login.defs.j2
        dest: /etc/login.defs
        owner: root
        group: root
        mode: '0644'
      when: "'CIS-4.y' not in compliance_exceptions"
      tags: [remediate, "CIS-4.y"]
```

> Used where the file should be fully owned/managed by the playbook rather than patched line-by-line, per the table's note.

---

### Task 4 of 11 — Set file permissions/ownership

| Row | Module |
|---|---|
| Set file permissions/ownership | `ansible.builtin.file` |

```yaml
    - name: "CIS-1.1.x - Remediate /tmp permissions"
      ansible.builtin.file:
        path: /tmp
        mode: '1777'
        state: directory
      when:
        - "'CIS-1.1.x' not in compliance_exceptions"
        - tmp_perms.stat.mode != '1777'
      tags: [remediate, "CIS-1.1.x"]

    - name: "CIS-1.2.x - Lock down /etc/shadow permissions"
      ansible.builtin.file:
        path: /etc/shadow
        owner: root
        group: shadow
        mode: '0640'
      when: "'CIS-1.2.x' not in compliance_exceptions"
      tags: [remediate, "CIS-1.2.x"]
```

---

### Task 5 of 11 — Remove or hold disallowed packages

| Row | Module |
|---|---|
| Remove or hold disallowed packages | `ansible.builtin.apt` (`state: absent`) / `ansible.builtin.dpkg_selections` (`selection: hold`) |

```yaml
    - name: "CIS-2.x - Remove disallowed telnet package"
      ansible.builtin.apt:
        name: telnet
        state: absent
      when:
        - "'CIS-2.x' not in compliance_exceptions"
        - telnet_present | default(false)
      tags: [remediate, "CIS-2.x"]

    - name: "CIS-2.x - Hold telnet from reinstallation"
      ansible.builtin.dpkg_selections:
        name: telnet
        selection: hold
      when: "'CIS-2.x' not in compliance_exceptions"
      tags: [remediate, "CIS-2.x"]
```

---

### Task 6 of 11 — Disable/mask unwanted services

| Row | Module |
|---|---|
| Disable/mask unwanted services | `ansible.builtin.systemd_service` (or `ansible.builtin.service`) |

```yaml
    - name: "CIS-2.y - Disable and mask avahi-daemon"
      ansible.builtin.systemd_service:
        name: avahi-daemon
        enabled: false
        masked: true
        state: stopped
      when:
        - "'CIS-2.y' not in compliance_exceptions"
        - avahi_enabled | default(false)
      tags: [remediate, "CIS-2.y"]
```

---

### Task 7 of 11 — Enforce sysctl/kernel parameters

| Row | Module |
|---|---|
| Enforce sysctl/kernel parameters | `ansible.builtin.sysctl` |

```yaml
    - name: "CIS-3.x - Enforce IP forwarding disabled"
      ansible.builtin.sysctl:
        name: net.ipv4.ip_forward
        value: '0'
        state: present
        reload: true
      when:
        - "'CIS-3.x' not in compliance_exceptions"
        - cis_3x_audit_result is failed
      tags: [remediate, "CIS-3.x"]
```

> Gated on the §4.1 Task 7 assert result — only remediates hosts that the audit already found non-compliant, keeping the run idempotent and evidence-driven rather than blindly reapplying.

---

### Task 8 of 11 — Enforce password policy (`login.defs`, PAM)

| Row | Module |
|---|---|
| Enforce password policy (`/etc/login.defs`, PAM) | `ansible.builtin.lineinfile` on `login.defs`; `ansible.builtin.pamd` |

```yaml
    - name: "CIS-4.z - Enforce PASS_MAX_DAYS in login.defs"
      ansible.builtin.lineinfile:
        path: /etc/login.defs
        regexp: '^PASS_MAX_DAYS'
        line: 'PASS_MAX_DAYS   365'
      when: "'CIS-4.z' not in compliance_exceptions"
      tags: [remediate, "CIS-4.z"]

    - name: "CIS-4.z - Enforce PAM password retry limit"
      ansible.builtin.pamd:
        name: common-auth
        type: auth
        control: required
        module_path: pam_tally2.so
        module_arguments: 'deny=5 onerr=fail'
        state: args_present
      when: "'CIS-4.z' not in compliance_exceptions"
      tags: [remediate, "CIS-4.z"]
```

> `pamd` is called out in the table as preferred over raw `lineinfile` on PAM files, since it edits PAM rules structurally rather than as free text.

---

### Task 9 of 11 — Manage user/group compliance

| Row | Module |
|---|---|
| Manage user/group compliance (e.g., lock unused accounts) | `ansible.builtin.user` |

```yaml
    - name: "CIS-4.x - Lock unused system account"
      ansible.builtin.user:
        name: "{{ unused_account_name | default('games') }}"
        password_lock: true
        shell: /usr/sbin/nologin
      when: "'CIS-4.x' not in compliance_exceptions"
      tags: [remediate, "CIS-4.x"]
```

---

### Task 10 of 11 — Configure firewall baseline (UFW)

| Row | Module |
|---|---|
| Configure firewall baseline (UFW) | `community.general.ufw` |

```yaml
    - name: "CIS-3.y - Set default deny incoming via UFW"
      community.general.ufw:
        default: deny
        direction: incoming
      when: "'CIS-3.y' not in compliance_exceptions"
      tags: [remediate, "CIS-3.y"]

    - name: "CIS-3.y - Allow SSH through UFW"
      community.general.ufw:
        rule: allow
        port: '22'
        proto: tcp
      when: "'CIS-3.y' not in compliance_exceptions"
      tags: [remediate, "CIS-3.y"]

    - name: "CIS-3.y - Enable UFW"
      community.general.ufw:
        state: enabled
      when: "'CIS-3.y' not in compliance_exceptions"
      tags: [remediate, "CIS-3.y"]
```

> Flagged in the table as the one common non-`ansible.builtin` dependency — worth confirming the `community.general` collection is already available in your execution environment before this task runs.

---

### Task 11 of 11 — Ensure auditd rules present, and restart/reload after remediation

| Row | Module |
|---|---|
| Ensure auditd rules present | `ansible.builtin.lineinfile` on `/etc/audit/rules.d/*.rules` + handler to reload auditd |
| Restart/reload service after remediation | `ansible.builtin.service` (as a **handler**) |

```yaml
    - name: "CIS-6.x - Ensure audit rule for /etc/passwd changes"
      ansible.builtin.lineinfile:
        path: /etc/audit/rules.d/cis-hardening.rules
        create: true
        mode: '0640'
        line: '-w /etc/passwd -p wa -k identity'
      notify: reload auditd
      when: "'CIS-6.x' not in compliance_exceptions"
      tags: [remediate, "CIS-6.x"]
```

Corresponding handlers (append to the `handlers:` section, not `tasks:`):

```yaml
  handlers:
    - name: restart sshd
      ansible.builtin.service:
        name: ssh
        state: restarted

    - name: reload auditd
      ansible.builtin.service:
        name: auditd
        state: restarted
```

> Both remediation tasks that touch live services (Task 1 and this one) `notify` a handler rather than restarting inline — this is what keeps remediation idempotent per §6.4: the restart only fires when a change actually occurred.

**§4.2 complete.** Validate with `--tags remediate` (drop `--check` once audit findings are confirmed) before moving to §4.3.

---

## §4.3 Verify (re-check after remediation) — task-by-task

### Task 1 of 4 — Re-run the same audit task post-remediation

| Row | Module |
|---|---|
| Re-run the same audit task post-remediation | Re-use the §4.1 check task |

```yaml
    - name: "CIS-5.2.10 - Verify PermitRootLogin after remediation"
      ansible.builtin.command: sshd -T
      register: sshd_verify
      changed_when: false
      when: "'CIS-5.2.10' not in compliance_exceptions"
      tags: [verify, "CIS-5.2.10"]
```

> Per the table's note: "verify" is structurally just re-invoking the same check logic used in §4.1, not a new mechanism — reuse rather than reinvent.

---

### Task 2 of 4 — Confirm no unintended breakage

| Row | Module |
|---|---|
| Confirm no unintended breakage | `ansible.builtin.wait_for` / `ansible.builtin.uri` |

```yaml
    - name: "CIS-5.2.10 - Confirm SSH still accepts connections post-hardening"
      ansible.builtin.wait_for:
        port: 22
        timeout: 30
      when: "'CIS-5.2.10' not in compliance_exceptions"
      tags: [verify, "CIS-5.2.10"]
```

> Since Task 1 of §4.2 touched `sshd_config` (the control most likely to break connectivity), this is the concrete case the table flags: confirm the service still accepts connections before the play ends, not just that the config line changed.

---

### Task 3 of 4 — Fail the run explicitly on unresolved non-compliance

| Row | Module |
|---|---|
| Fail the run explicitly on unresolved non-compliance | `ansible.builtin.fail` |

```yaml
    - name: "CIS-5.2.10 - Fail explicitly if still non-compliant"
      ansible.builtin.fail:
        msg: "CIS-5.2.10 remediation did not take effect: PermitRootLogin still permitted"
      when:
        - "'CIS-5.2.10' not in compliance_exceptions"
        - "'permitrootlogin yes' in (sshd_verify.stdout | lower)"
      tags: [verify, "CIS-5.2.10"]
```

> Distinguishes "control could not be remediated" (this task) from "control not applicable" (the `compliance_exceptions` skip condition) — per the table's note, these are two different states and should never be conflated in reporting.

---

### Task 4 of 4 — Skip a control intentionally (documented exception)

| Row | Module |
|---|---|
| Skip a control intentionally (documented exception) | `when:` condition against a `compliance_exceptions` variable |

This isn't a new task so much as the condition already threaded through every task above (`when: "'<CONTROL-ID>' not in compliance_exceptions"`). Making it explicit as its own row/step:

```yaml
  vars:
    compliance_exceptions:
      - "CIS-3.y"   # Example: UFW deferred, host sits behind an upstream firewall appliance
```

> Per §6.3's design principle: exceptions live in `group_vars`/`host_vars` as data, not as commented-out tasks — this keeps them visible, reviewable, and auditable rather than tribal knowledge.

**§4.3 complete.** Validate with `--tags verify` and confirm the `fail` task correctly distinguishes real failures from documented exceptions, per the Section 7 exercise, before moving on to §4.4 (Reporting/Evidence).

---

## §5. Assembled playbook after §4.1–§4.3

```yaml
---
- name: Ubuntu CIS-Style Hardening Playbook (sample controls)
  hosts: hardening_targets
  become: true
  vars:
    compliance_exceptions:
      - "CIS-3.y"
    compliance_results: []

  tasks:
    # --- §4.1 Audit / Check (read-only) ---
    - name: "CIS-5.2.10 - Check current PermitRootLogin line"
      ansible.builtin.slurp:
        src: /etc/ssh/sshd_config
      register: sshd_config_raw
      when: "'CIS-5.2.10' not in compliance_exceptions"
      tags: [audit, "CIS-5.2.10"]

    - name: "CIS-1.1.x - Check /tmp permissions"
      ansible.builtin.stat:
        path: /tmp
      register: tmp_perms
      when: "'CIS-1.1.x' not in compliance_exceptions"
      tags: [audit, "CIS-1.1.x"]

    - name: "CIS-2.x - Gather package facts"
      ansible.builtin.package_facts:
      tags: [audit, "CIS-2.x"]

    - name: "CIS-2.x - Check for disallowed telnet package"
      ansible.builtin.set_fact:
        telnet_present: "{{ 'telnet' in ansible_facts.packages }}"
      when: "'CIS-2.x' not in compliance_exceptions"
      tags: [audit, "CIS-2.x"]

    - name: "CIS-2.y - Gather service facts"
      ansible.builtin.service_facts:
      tags: [audit, "CIS-2.y"]

    - name: "CIS-2.y - Check for unwanted avahi-daemon service"
      ansible.builtin.set_fact:
        avahi_enabled: "{{ ansible_facts.services['avahi-daemon.service'].status | default('not-found') == 'enabled' }}"
      when: "'CIS-2.y' not in compliance_exceptions"
      tags: [audit, "CIS-2.y"]

    - name: "CIS-3.x - Check current ip_forward value"
      ansible.builtin.command: sysctl -n net.ipv4.ip_forward
      register: ip_forward_check
      changed_when: false
      when: "'CIS-3.x' not in compliance_exceptions"
      tags: [audit, "CIS-3.x"]

    - name: "CIS-4.x - Look up account entry via NSS"
      ansible.builtin.getent:
        database: passwd
        key: "{{ unused_account_name | default('games') }}"
      when: "'CIS-4.x' not in compliance_exceptions"
      tags: [audit, "CIS-4.x"]

    - name: "CIS-3.x - Assert ip_forward is disabled"
      ansible.builtin.assert:
        that: ip_forward_check.stdout | trim == '0'
        fail_msg: "CIS-3.x non-compliant: net.ipv4.ip_forward is enabled"
        success_msg: "CIS-3.x compliant: net.ipv4.ip_forward is disabled"
      when: "'CIS-3.x' not in compliance_exceptions"
      register: cis_3x_audit_result
      ignore_errors: true
      tags: [audit, "CIS-3.x"]

    # --- §4.2 Remediation ---
    - name: "CIS-5.2.10 - Remediate PermitRootLogin"
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart sshd
      when:
        - "'CIS-5.2.10' not in compliance_exceptions"
        - "'permitrootlogin yes' in (sshd_config_raw.content | b64decode | lower)"
      tags: [remediate, "CIS-5.2.10"]

    - name: "CIS-5.x - Enforce SSH hardening block"
      ansible.builtin.blockinfile:
        path: /etc/ssh/sshd_config.d/99-cis-hardening.conf
        create: true
        mode: '0644'
        block: |
          Protocol 2
          X11Forwarding no
          MaxAuthTries 4
      notify: restart sshd
      when: "'CIS-5.x' not in compliance_exceptions"
      tags: [remediate, "CIS-5.x"]

    - name: "CIS-4.y - Deploy fully-managed login.defs"
      ansible.builtin.template:
        src: templates/login.defs.j2
        dest: /etc/login.defs
        owner: root
        group: root
        mode: '0644'
      when: "'CIS-4.y' not in compliance_exceptions"
      tags: [remediate, "CIS-4.y"]

    - name: "CIS-1.1.x - Remediate /tmp permissions"
      ansible.builtin.file:
        path: /tmp
        mode: '1777'
        state: directory
      when:
        - "'CIS-1.1.x' not in compliance_exceptions"
        - tmp_perms.stat.mode != '1777'
      tags: [remediate, "CIS-1.1.x"]

    - name: "CIS-1.2.x - Lock down /etc/shadow permissions"
      ansible.builtin.file:
        path: /etc/shadow
        owner: root
        group: shadow
        mode: '0640'
      when: "'CIS-1.2.x' not in compliance_exceptions"
      tags: [remediate, "CIS-1.2.x"]

    - name: "CIS-2.x - Remove disallowed telnet package"
      ansible.builtin.apt:
        name: telnet
        state: absent
      when:
        - "'CIS-2.x' not in compliance_exceptions"
        - telnet_present | default(false)
      tags: [remediate, "CIS-2.x"]

    - name: "CIS-2.x - Hold telnet from reinstallation"
      ansible.builtin.dpkg_selections:
        name: telnet
        selection: hold
      when: "'CIS-2.x' not in compliance_exceptions"
      tags: [remediate, "CIS-2.x"]

    - name: "CIS-2.y - Disable and mask avahi-daemon"
      ansible.builtin.systemd_service:
        name: avahi-daemon
        enabled: false
        masked: true
        state: stopped
      when:
        - "'CIS-2.y' not in compliance_exceptions"
        - avahi_enabled | default(false)
      tags: [remediate, "CIS-2.y"]

    - name: "CIS-3.x - Enforce IP forwarding disabled"
      ansible.builtin.sysctl:
        name: net.ipv4.ip_forward
        value: '0'
        state: present
        reload: true
      when:
        - "'CIS-3.x' not in compliance_exceptions"
        - cis_3x_audit_result is failed
      tags: [remediate, "CIS-3.x"]

    - name: "CIS-4.z - Enforce PASS_MAX_DAYS in login.defs"
      ansible.builtin.lineinfile:
        path: /etc/login.defs
        regexp: '^PASS_MAX_DAYS'
        line: 'PASS_MAX_DAYS   365'
      when: "'CIS-4.z' not in compliance_exceptions"
      tags: [remediate, "CIS-4.z"]

    - name: "CIS-4.z - Enforce PAM password retry limit"
      ansible.builtin.pamd:
        name: common-auth
        type: auth
        control: required
        module_path: pam_tally2.so
        module_arguments: 'deny=5 onerr=fail'
        state: args_present
      when: "'CIS-4.z' not in compliance_exceptions"
      tags: [remediate, "CIS-4.z"]

    - name: "CIS-4.x - Lock unused system account"
      ansible.builtin.user:
        name: "{{ unused_account_name | default('games') }}"
        password_lock: true
        shell: /usr/sbin/nologin
      when: "'CIS-4.x' not in compliance_exceptions"
      tags: [remediate, "CIS-4.x"]

    - name: "CIS-3.y - Set default deny incoming via UFW"
      community.general.ufw:
        default: deny
        direction: incoming
      when: "'CIS-3.y' not in compliance_exceptions"
      tags: [remediate, "CIS-3.y"]

    - name: "CIS-3.y - Allow SSH through UFW"
      community.general.ufw:
        rule: allow
        port: '22'
        proto: tcp
      when: "'CIS-3.y' not in compliance_exceptions"
      tags: [remediate, "CIS-3.y"]

    - name: "CIS-3.y - Enable UFW"
      community.general.ufw:
        state: enabled
      when: "'CIS-3.y' not in compliance_exceptions"
      tags: [remediate, "CIS-3.y"]

    - name: "CIS-6.x - Ensure audit rule for /etc/passwd changes"
      ansible.builtin.lineinfile:
        path: /etc/audit/rules.d/cis-hardening.rules
        create: true
        mode: '0640'
        line: '-w /etc/passwd -p wa -k identity'
      notify: reload auditd
      when: "'CIS-6.x' not in compliance_exceptions"
      tags: [remediate, "CIS-6.x"]

    # --- §4.3 Verify ---
    - name: "CIS-5.2.10 - Verify PermitRootLogin after remediation"
      ansible.builtin.command: sshd -T
      register: sshd_verify
      changed_when: false
      when: "'CIS-5.2.10' not in compliance_exceptions"
      tags: [verify, "CIS-5.2.10"]

    - name: "CIS-5.2.10 - Confirm SSH still accepts connections post-hardening"
      ansible.builtin.wait_for:
        port: 22
        timeout: 30
      when: "'CIS-5.2.10' not in compliance_exceptions"
      tags: [verify, "CIS-5.2.10"]

    - name: "CIS-5.2.10 - Fail explicitly if still non-compliant"
      ansible.builtin.fail:
        msg: "CIS-5.2.10 remediation did not take effect: PermitRootLogin still permitted"
      when:
        - "'CIS-5.2.10' not in compliance_exceptions"
        - "'permitrootlogin yes' in (sshd_verify.stdout | lower)"
      tags: [verify, "CIS-5.2.10"]

  handlers:
    - name: restart sshd
      ansible.builtin.service:
        name: ssh
        state: restarted

    - name: reload auditd
      ansible.builtin.service:
        name: auditd
        state: restarted
```

> **Note on task ordering / dependencies, surfaced by building row-by-row:**
> - Task 7 of §4.2 (sysctl remediation) depends on `cis_3x_audit_result`, which is only created by Task 7 of §4.1 — appending strictly row-by-row across sections still requires audit tasks to run before their matching remediation, which they do here since §4.1 is appended first in full.
> - `CIS-3.y` (UFW) is listed in `compliance_exceptions` above purely as a **worked example** of §4.3 Task 4 (documented exception) — remove it once your group has made a real decision on firewall ownership per §3.2 point 4.
> - Handlers accumulate across both remediation tasks that touch live services (SSH config, auditd rules) — as more controls are appended in a real build, keep watching for handler name collisions.

---

## Next steps

- Append §4.4 (Reporting/Evidence) the same way, row by row: build the structured per-control result (`set_fact` + `template`), then the aggregation (`fetch`) and notification (`uri` webhook) tasks.
- Once the hand-rolled version above is validated on 2–3 controls, revisit §4.5's build-vs-adopt note — decide whether to keep extending this file control-by-control or graduate to a maintained collection (e.g., `ansible-lockdown.UBUNTU22-CIS`) with these as org-specific overlays.
- Carry the ordering notes above into the Gaps & Action Items template (§8 of the workshop doc).
