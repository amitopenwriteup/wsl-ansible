# Ubuntu Commissioning & Decommissioning Playbooks — Incremental Build (Task-by-Task)

**Companion to:** *Workshop: Server Provisioning & Commissioning/Decommissioning for Ubuntu (via Ansible)*
**Purpose:** Build the commissioning playbook (§4.1 Provisioning → §4.2 Commissioning → §4.3 Verify) and the decommissioning playbook (§4.4) one task at a time, in the exact order the rows appear in each module table, so each addition can be reviewed against the one real server before the next is appended.

---

## How to use this document

- Each task below corresponds to exactly one row of the module table in the referenced section.
- Tasks are appended in row order — do not skip ahead.
- Per §7's exercise guidance: run commissioning tasks against a throwaway test VM or in `--check` mode; **do not execute** decommissioning tasks against the real server — read/validate them line by line instead.
- The two assembled playbooks (§5.1 after §4.1–§4.3, §5.2 after §4.4) mirror the two skeletons in the workshop doc.

---

## §4.1 Provisioning (getting to "reachable") — task-by-task

### Task 1 of 6 — Confirm host is reachable/SSH-ready

| Row | Module |
|---|---|
| Confirm host is reachable/SSH-ready | `ansible.builtin.ping` |

```yaml
---
- name: Commission Ubuntu Server
  hosts: new_servers
  become: true
  vars:
    baseline_packages:
      - chrony
      - fail2ban
      - unattended-upgrades
    target_hostname: "{{ inventory_hostname }}"
    target_timezone: "Etc/UTC"
    cmdb_integration_enabled: false

  tasks:
    - name: Confirm host is reachable
      ansible.builtin.ping:
      tags: [provision]
```

> The table calls this out as the standard first task in any provisioning/commissioning play — nothing else should run before this succeeds.

---

### Task 2 of 6 — Gather baseline facts

| Row | Module |
|---|---|
| Gather baseline facts | `ansible.builtin.setup` |

```yaml
    - name: Gather baseline facts
      ansible.builtin.setup:
      tags: [provision]
```

> Confirms OS version/architecture/memory match expectations before anything else proceeds — the same rationale as the pre-flight fact-gathering task in the patch management workshop.

---

### Task 3 of 6 — Wait for SSH to come up (if freshly booted)

| Row | Module |
|---|---|
| Wait for SSH to come up (if freshly booted) | `ansible.builtin.wait_for_connection` |

```yaml
    - name: Wait for SSH connection to stabilize
      ansible.builtin.wait_for_connection:
        timeout: 120
      when: freshly_booted | default(false)
      tags: [provision]
```

> Table's note: less relevant for the one manually-provisioned server today, so it's gated behind `freshly_booted` (default off) rather than removed — designed in now for when provisioning becomes cloud/VM API-driven later.

---

### Task 4 of 6 — Set hostname

| Row | Module |
|---|---|
| Set hostname | `ansible.builtin.hostname` |

```yaml
    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ target_hostname }}"
      tags: [provision]
```

---

### Task 5 of 6 — Set timezone

| Row | Module |
|---|---|
| Set timezone | `ansible.builtin.timezone` |

```yaml
    - name: Set timezone
      ansible.builtin.timezone:
        name: "{{ target_timezone }}"
      tags: [provision]
```

> Flagged in the table as often overlooked — worth confirming this matches whatever timezone the hardening/auditd logging from the compliance workshop assumes, so log timestamps line up across hosts later.

---

### Task 6 of 6 — Configure `/etc/hosts` entries

| Row | Module |
|---|---|
| Configure `/etc/hosts` entries | `ansible.builtin.lineinfile` or `ansible.builtin.template` |

```yaml
    - name: Configure /etc/hosts entries
      ansible.builtin.template:
        src: templates/hosts.j2
        dest: /etc/hosts
        owner: root
        group: root
        mode: '0644'
      when: hosts_file_managed | default(false)
      tags: [provision]
```

> Table's note: "especially relevant once there's more than one host to resolve" — with one server this is close to a no-op, so it's gated behind `hosts_file_managed` rather than forced, but the task exists now so it's ready when a second host shows up.

**§4.1 complete.** Validate with `--tags provision` against the test target before moving to §4.2.

---

## §4.2 Commissioning (making it service-ready) — task-by-task

### Task 1 of 8 — Install baseline package set

| Row | Module |
|---|---|
| Install baseline package set | `ansible.builtin.apt` (`name: [...]`, `state: present`) |

```yaml
    - name: Install baseline packages
      ansible.builtin.apt:
        name: "{{ baseline_packages }}"
        state: present
        update_cache: true
      tags: [commission]
```

> Same externalize-the-list pattern flagged in the table as matching the patch playbook's `upgrade_mode` approach — `baseline_packages` lives in `vars`/`group_vars`, not hardcoded per host.

---

### Task 2 of 8 — Create standard users/groups

| Row | Module |
|---|---|
| Create standard users/groups | `ansible.builtin.user` / `ansible.builtin.group` |

```yaml
    - name: Create standard admin group
      ansible.builtin.group:
        name: admins
        state: present
      tags: [commission]

    - name: Create standard admin user
      ansible.builtin.user:
        name: admin
        groups: admins
        shell: /bin/bash
        create_home: true
      tags: [commission]
```

---

### Task 3 of 8 — Deploy SSH keys for authorized access

| Row | Module |
|---|---|
| Deploy SSH keys for authorized access | `ansible.builtin.authorized_key` |

```yaml
    - name: Deploy admin SSH key
      ansible.builtin.authorized_key:
        user: admin
        state: present
        key: "{{ lookup('file', 'files/admin_id_ed25519.pub') }}"
      tags: [commission]
```

---

### Task 4 of 8 — Apply hardening baseline

| Row | Module |
|---|---|
| Apply hardening baseline | Include/import the hardening playbook or role from the compliance workshop (`ansible.builtin.import_playbook` / `ansible.builtin.include_role`) |

```yaml
    - name: Apply hardening baseline
      ansible.builtin.import_playbook: hardening.yml
      tags: [commission, hardening]
```

> This is the key integration point called out in the table: commissioning **calls** the CIS-hardening playbook built in the compliance workshop rather than duplicating its check→remediate→verify tasks here. Confirm the file path matches your actual hardening playbook's location before running.

---

### Task 5 of 8 — Apply patch baseline

| Row | Module |
|---|---|
| Apply patch baseline | Include/import the patch management playbook |

```yaml
    - name: Apply patch baseline
      ansible.builtin.import_playbook: patch_management.yml
      tags: [commission, patching]
```

> Same reuse principle as Task 4 — new servers should come up already at current patch level via the existing patching automation, not a parallel upgrade task written here.

---

### Task 6 of 8 — Install monitoring/logging agent

| Row | Module |
|---|---|
| Install monitoring/logging agent | `ansible.builtin.apt` (agent package) + `ansible.builtin.template` (agent config) + `ansible.builtin.service` (enable/start) |

```yaml
    - name: Install monitoring agent package
      ansible.builtin.apt:
        name: "{{ monitoring_agent_package | default('prometheus-node-exporter') }}"
        state: present
      tags: [commission]

    - name: Deploy monitoring agent config
      ansible.builtin.template:
        src: templates/monitoring_agent.conf.j2
        dest: /etc/monitoring_agent/config.yml
        mode: '0644'
      notify: restart monitoring agent
      tags: [commission]

    - name: Enable and start monitoring agent
      ansible.builtin.service:
        name: "{{ monitoring_agent_service | default('prometheus-node-exporter') }}"
        enabled: true
        state: started
      tags: [commission]
```

---

### Task 7 of 8 — Configure log forwarding / rsyslog

| Row | Module |
|---|---|
| Configure log forwarding / rsyslog | `ansible.builtin.template` on rsyslog config + `ansible.builtin.service` (restart via handler) |

```yaml
    - name: Deploy rsyslog forwarding config
      ansible.builtin.template:
        src: templates/rsyslog_forwarding.conf.j2
        dest: /etc/rsyslog.d/60-forwarding.conf
        mode: '0644'
      notify: restart rsyslog
      tags: [commission]
```

Handler (append to `handlers:`):

```yaml
  handlers:
    - name: restart monitoring agent
      ansible.builtin.service:
        name: "{{ monitoring_agent_service | default('prometheus-node-exporter') }}"
        state: restarted

    - name: restart rsyslog
      ansible.builtin.service:
        name: rsyslog
        state: restarted
```

> Ensures logs from this host land in the same central place as everything else from day one — same `notify`-a-handler idempotency pattern used throughout the hardening workshop.

---

### Task 8 of 8 — Register host in internal inventory/CMDB

| Row | Module |
|---|---|
| Register host in internal inventory/CMDB | `ansible.builtin.uri` (POST to an internal API/webhook) |

```yaml
    - name: Register host as commissioned in CMDB
      ansible.builtin.uri:
        url: "https://cmdb.internal.example.com/api/hosts/{{ target_hostname }}"
        method: POST
        body_format: json
        body:
          status: commissioning-in-progress
      when: cmdb_integration_enabled | default(false)
      tags: [commission]
```

> Gated behind `cmdb_integration_enabled` (default `false`) per §6.6's standardization point — runs cleanly today with no CMDB, lights up later without editing this task.

**§4.2 complete.** Validate with `--tags commission --check` and confirm the `import_playbook` file paths resolve correctly before moving to §4.3.

---

## §4.3 Verify (readiness gate before "in service") — task-by-task

### Task 1 of 5 — Confirm expected services are running

| Row | Module |
|---|---|
| Confirm expected services are running | `ansible.builtin.service_facts` |

```yaml
    - name: Gather service facts for readiness check
      ansible.builtin.service_facts:
      tags: [verify]

    - name: Assert monitoring agent service is running
      ansible.builtin.assert:
        that: "ansible_facts.services[monitoring_agent_service | default('prometheus-node-exporter') + '.service'].state == 'running'"
        fail_msg: "Monitoring agent is not running post-commissioning"
      tags: [verify]
```

---

### Task 2 of 5 — Confirm expected ports/endpoints respond

| Row | Module |
|---|---|
| Confirm expected ports/endpoints respond | `ansible.builtin.wait_for` / `ansible.builtin.uri` |

```yaml
    - name: Verify SSH service is active post-hardening
      ansible.builtin.wait_for:
        port: 22
        timeout: 30
      tags: [verify]
```

> Same health-check pattern flagged in the table as reused directly from the patch management workshop.

---

### Task 3 of 5 — Confirm hardening controls landed

| Row | Module |
|---|---|
| Confirm hardening controls landed | Re-run the compliance-as-code check tasks |

```yaml
    - name: Re-run hardening audit checks as commissioning verify step
      ansible.builtin.import_playbook: hardening.yml
      vars:
        run_mode: audit_only
      tags: [verify, hardening]
```

> Per the table's note, this is deliberately *reuse*, not a rebuilt check — commissioning's "verify" step for hardening **is** the compliance workshop's audit step, invoked with an audit-only variable rather than re-implemented here.

---

### Task 4 of 5 — Fail commissioning on unmet readiness

| Row | Module |
|---|---|
| Fail commissioning on unmet readiness | `ansible.builtin.fail` |

```yaml
    - name: Fail commissioning if readiness checks did not pass
      ansible.builtin.fail:
        msg: "Commissioning readiness checks failed for {{ target_hostname }} — see prior task results"
      when: commissioning_readiness_failed | default(false)
      tags: [verify]
```

> Prevents a broken host from silently being marked "in service" — mirrors the explicit-fail pattern from both the patch and hardening workshops.

---

### Task 5 of 5 — Tag/label host as commissioned

| Row | Module |
|---|---|
| Tag/label host as commissioned | `ansible.builtin.uri` (update CMDB/inventory record) or `ansible.builtin.set_fact` + report |

```yaml
    - name: Register host as commissioned in CMDB
      ansible.builtin.uri:
        url: "https://cmdb.internal.example.com/api/hosts/{{ target_hostname }}"
        method: POST
        body_format: json
        body:
          status: commissioned
          commissioned_date: "{{ ansible_date_time.iso8601 }}"
      when: cmdb_integration_enabled | default(false)
      tags: [verify]
```

> Gives an explicit "completed commissioning on {date}" record — this is the same CMDB call pattern as §4.2 Task 8, now marking final status rather than in-progress.

**§4.3 complete.** Validate with `--tags verify` and confirm the readiness `fail` task correctly gates a host from being marked commissioned, per the Section 7 exercise, before moving to §4.4 (decommissioning — built and reviewed, **not executed**, against the real server).

---

## §5.1 Assembled commissioning playbook after §4.1–§4.3

```yaml
---
- name: Commission Ubuntu Server
  hosts: new_servers
  become: true
  vars:
    baseline_packages:
      - chrony
      - fail2ban
      - unattended-upgrades
    target_hostname: "{{ inventory_hostname }}"
    target_timezone: "Etc/UTC"
    cmdb_integration_enabled: false

  tasks:
    # --- §4.1 Provisioning ---
    - name: Confirm host is reachable
      ansible.builtin.ping:
      tags: [provision]

    - name: Gather baseline facts
      ansible.builtin.setup:
      tags: [provision]

    - name: Wait for SSH connection to stabilize
      ansible.builtin.wait_for_connection:
        timeout: 120
      when: freshly_booted | default(false)
      tags: [provision]

    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ target_hostname }}"
      tags: [provision]

    - name: Set timezone
      ansible.builtin.timezone:
        name: "{{ target_timezone }}"
      tags: [provision]

    - name: Configure /etc/hosts entries
      ansible.builtin.template:
        src: templates/hosts.j2
        dest: /etc/hosts
        owner: root
        group: root
        mode: '0644'
      when: hosts_file_managed | default(false)
      tags: [provision]

    # --- §4.2 Commissioning ---
    - name: Install baseline packages
      ansible.builtin.apt:
        name: "{{ baseline_packages }}"
        state: present
        update_cache: true
      tags: [commission]

    - name: Create standard admin group
      ansible.builtin.group:
        name: admins
        state: present
      tags: [commission]

    - name: Create standard admin user
      ansible.builtin.user:
        name: admin
        groups: admins
        shell: /bin/bash
        create_home: true
      tags: [commission]

    - name: Deploy admin SSH key
      ansible.builtin.authorized_key:
        user: admin
        state: present
        key: "{{ lookup('file', 'files/admin_id_ed25519.pub') }}"
      tags: [commission]

    - name: Apply hardening baseline
      ansible.builtin.import_playbook: hardening.yml
      tags: [commission, hardening]

    - name: Apply patch baseline
      ansible.builtin.import_playbook: patch_management.yml
      tags: [commission, patching]

    - name: Install monitoring agent package
      ansible.builtin.apt:
        name: "{{ monitoring_agent_package | default('prometheus-node-exporter') }}"
        state: present
      tags: [commission]

    - name: Deploy monitoring agent config
      ansible.builtin.template:
        src: templates/monitoring_agent.conf.j2
        dest: /etc/monitoring_agent/config.yml
        mode: '0644'
      notify: restart monitoring agent
      tags: [commission]

    - name: Enable and start monitoring agent
      ansible.builtin.service:
        name: "{{ monitoring_agent_service | default('prometheus-node-exporter') }}"
        enabled: true
        state: started
      tags: [commission]

    - name: Deploy rsyslog forwarding config
      ansible.builtin.template:
        src: templates/rsyslog_forwarding.conf.j2
        dest: /etc/rsyslog.d/60-forwarding.conf
        mode: '0644'
      notify: restart rsyslog
      tags: [commission]

    - name: Register host as commissioned in CMDB (in-progress)
      ansible.builtin.uri:
        url: "https://cmdb.internal.example.com/api/hosts/{{ target_hostname }}"
        method: POST
        body_format: json
        body:
          status: commissioning-in-progress
      when: cmdb_integration_enabled | default(false)
      tags: [commission]

    # --- §4.3 Verify ---
    - name: Gather service facts for readiness check
      ansible.builtin.service_facts:
      tags: [verify]

    - name: Assert monitoring agent service is running
      ansible.builtin.assert:
        that: "ansible_facts.services[monitoring_agent_service | default('prometheus-node-exporter') + '.service'].state == 'running'"
        fail_msg: "Monitoring agent is not running post-commissioning"
      tags: [verify]

    - name: Verify SSH service is active post-hardening
      ansible.builtin.wait_for:
        port: 22
        timeout: 30
      tags: [verify]

    - name: Re-run hardening audit checks as commissioning verify step
      ansible.builtin.import_playbook: hardening.yml
      vars:
        run_mode: audit_only
      tags: [verify, hardening]

    - name: Fail commissioning if readiness checks did not pass
      ansible.builtin.fail:
        msg: "Commissioning readiness checks failed for {{ target_hostname }} — see prior task results"
      when: commissioning_readiness_failed | default(false)
      tags: [verify]

    - name: Register host as commissioned in CMDB (final)
      ansible.builtin.uri:
        url: "https://cmdb.internal.example.com/api/hosts/{{ target_hostname }}"
        method: POST
        body_format: json
        body:
          status: commissioned
          commissioned_date: "{{ ansible_date_time.iso8601 }}"
      when: cmdb_integration_enabled | default(false)
      tags: [verify]

  handlers:
    - name: restart monitoring agent
      ansible.builtin.service:
        name: "{{ monitoring_agent_service | default('prometheus-node-exporter') }}"
        state: restarted

    - name: restart rsyslog
      ansible.builtin.service:
        name: rsyslog
        state: restarted
```

> **Note surfaced by building row-by-row:** two CMDB `uri` calls now exist — one marking `commissioning-in-progress` (§4.2 Task 8) and one marking `commissioned` (§4.3 Task 5). Confirm your actual CMDB schema supports both states, or collapse to a single final-status call if the in-progress state adds no value for your team.

---

## §4.4 Decommissioning — task-by-task

**Per §7's exercise guidance: build and read through these against the real server's actual service/data/credential inventory — do not execute the `stop`/`revoke`/`delete` tasks against it.**

### Task 1 of 8 — Drain/stop application services gracefully

| Row | Module |
|---|---|
| Drain/stop application services gracefully | `ansible.builtin.service` (`state: stopped`) |

```yaml
---
- name: Decommission Ubuntu Server
  hosts: decommission_targets
  become: true
  vars:
    backup_dest_dir: /var/backups/decommission
    services_to_stop:
      - myapp
    cmdb_integration_enabled: false
    lb_integration_enabled: false
    monitoring_integration_enabled: false

  tasks:
    - name: Stop application services gracefully
      ansible.builtin.service:
        name: "{{ item }}"
        state: stopped
      loop: "{{ services_to_stop }}"
      tags: [decommission]
```

> Table's note: order matters — app-level services stop before anything else in this play, so nothing downstream (monitoring, CMDB) is deregistering a host that's still actively serving traffic.

---

### Task 2 of 8 — Notify dependent systems (load balancer, DNS, monitoring)

| Row | Module |
|---|---|
| Notify dependent systems (load balancer, DNS, monitoring) | `ansible.builtin.uri` |

```yaml
    - name: Notify load balancer to deregister host
      ansible.builtin.uri:
        url: "https://lb.internal.example.com/api/deregister/{{ inventory_hostname }}"
        method: POST
      when: lb_integration_enabled | default(false)
      tags: [decommission]
```

---

### Task 3 of 8 — Back up final data/logs before teardown

| Row | Module |
|---|---|
| Back up final data/logs before teardown | `ansible.builtin.archive` + `ansible.builtin.fetch` |

```yaml
    - name: Archive final logs and data before teardown
      ansible.builtin.archive:
        path:
          - /var/log
          - /etc
        dest: "{{ backup_dest_dir }}/{{ inventory_hostname }}_final_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
      tags: [decommission]

    - name: Fetch final archive to control node
      ansible.builtin.fetch:
        src: "{{ backup_dest_dir }}/{{ inventory_hostname }}_final_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
        dest: "backups/decommissioned/"
        flat: true
      tags: [decommission]
```

> Table's note: same `archive` + `fetch` pattern as the pre-patch backup step from the patch management workshop, reused rather than reinvented.

---

### Task 4 of 8 — Revoke SSH keys / disable accounts

| Row | Module |
|---|---|
| Revoke SSH keys / disable accounts | `ansible.builtin.authorized_key` (`state: absent`) / `ansible.builtin.user` (`state: absent` or lock) |

```yaml
    - name: Revoke admin SSH key
      ansible.builtin.authorized_key:
        user: admin
        state: absent
        key: "{{ lookup('file', 'files/admin_id_ed25519.pub') }}"
      tags: [decommission]

    - name: Lock admin account
      ansible.builtin.user:
        name: admin
        password_lock: true
        shell: /usr/sbin/nologin
      tags: [decommission]
```

> Table's note: explicit credential teardown, not just "the host is gone so it doesn't matter" — important if backups (Task 3) or logs referencing this host persist elsewhere after teardown.

---

### Task 5 of 8 — Remove monitoring/logging agent registration

| Row | Module |
|---|---|
| Remove monitoring/logging agent registration | `ansible.builtin.uri` (deregister from monitoring platform) |

```yaml
    - name: Deregister host from monitoring platform
      ansible.builtin.uri:
        url: "https://monitoring.internal.example.com/api/hosts/{{ inventory_hostname }}"
        method: DELETE
      when: monitoring_integration_enabled | default(false)
      tags: [decommission]
```

> Table's note: prevents stale "host down" alerts firing forever after teardown — easy to forget since the monitoring agent itself was installed by commissioning (§4.2 Task 6) but nothing tears it down without this explicit step.

---

### Task 6 of 8 — Remove from internal inventory/CMDB

| Row | Module |
|---|---|
| Remove from internal inventory/CMDB | `ansible.builtin.uri` (DELETE/PATCH to inventory API) |

```yaml
    - name: Deregister host from CMDB
      ansible.builtin.uri:
        url: "https://cmdb.internal.example.com/api/hosts/{{ inventory_hostname }}"
        method: DELETE
      when: cmdb_integration_enabled | default(false)
      tags: [decommission]
```

> Mirrors the registration call from §4.2 Task 8 / §4.3 Task 5 — same API, opposite direction, as the table notes.

---

### Task 7 of 8 — Wipe sensitive data before host is powered off/reclaimed

| Row | Module |
|---|---|
| Wipe sensitive data before host is powered off/reclaimed | `ansible.builtin.shell` (secure wipe command) *or* rely on infra-level disk destruction if cloud/VM |

```yaml
    - name: Securely wipe sensitive data directories (policy decision required)
      ansible.builtin.shell: shred -uz -n 3 {{ item }}
      loop: "{{ secure_wipe_paths | default([]) }}"
      when:
        - secure_wipe_enabled | default(false)
        - secure_wipe_paths is defined
      tags: [decommission]
```

> Flagged explicitly in the table as a **policy decision, not a default** — left off (`secure_wipe_enabled: false`) until the team decides how sensitive this host's data is and whether infra-level disk destruction (not applicable for bare metal) covers it instead.

---

### Task 8 of 8 — Power off / stop the instance (if VM/cloud)

| Row | Module |
|---|---|
| Power off / stop the instance (if VM/cloud) | Platform-specific module (e.g., cloud provider collection) — **not built-in for bare metal** |

```yaml
    - name: Confirm decommission checklist complete
      ansible.builtin.debug:
        msg: "Host {{ inventory_hostname }} decommissioned. Manual step remaining: power off / reclaim hardware."
      tags: [decommission]
```

> Table's note: for the single physical server this workshop is scoped to, this step is explicitly **not** automated — called out as a manual action rather than scripted around, since there's no bare-metal built-in for it. If/when a cloud provider is introduced later, this task becomes the one to replace with that provider's collection module.

**§4.4 complete.** Per §7: read through the full assembled playbook below against the real server's actual service list, data locations, and credentials — confirm nothing is missing from the checklist — without running the `stop`/`revoke`/`shred`/`DELETE` tasks against it.

---

## §5.2 Assembled decommissioning playbook after §4.4

```yaml
---
- name: Decommission Ubuntu Server
  hosts: decommission_targets
  become: true
  vars:
    backup_dest_dir: /var/backups/decommission
    services_to_stop:
      - myapp
    cmdb_integration_enabled: false
    lb_integration_enabled: false
    monitoring_integration_enabled: false
    secure_wipe_enabled: false

  tasks:
    - name: Stop application services gracefully
      ansible.builtin.service:
        name: "{{ item }}"
        state: stopped
      loop: "{{ services_to_stop }}"
      tags: [decommission]

    - name: Notify load balancer to deregister host
      ansible.builtin.uri:
        url: "https://lb.internal.example.com/api/deregister/{{ inventory_hostname }}"
        method: POST
      when: lb_integration_enabled | default(false)
      tags: [decommission]

    - name: Archive final logs and data before teardown
      ansible.builtin.archive:
        path:
          - /var/log
          - /etc
        dest: "{{ backup_dest_dir }}/{{ inventory_hostname }}_final_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
      tags: [decommission]

    - name: Fetch final archive to control node
      ansible.builtin.fetch:
        src: "{{ backup_dest_dir }}/{{ inventory_hostname }}_final_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
        dest: "backups/decommissioned/"
        flat: true
      tags: [decommission]

    - name: Revoke admin SSH key
      ansible.builtin.authorized_key:
        user: admin
        state: absent
        key: "{{ lookup('file', 'files/admin_id_ed25519.pub') }}"
      tags: [decommission]

    - name: Lock admin account
      ansible.builtin.user:
        name: admin
        password_lock: true
        shell: /usr/sbin/nologin
      tags: [decommission]

    - name: Deregister host from monitoring platform
      ansible.builtin.uri:
        url: "https://monitoring.internal.example.com/api/hosts/{{ inventory_hostname }}"
        method: DELETE
      when: monitoring_integration_enabled | default(false)
      tags: [decommission]

    - name: Deregister host from CMDB
      ansible.builtin.uri:
        url: "https://cmdb.internal.example.com/api/hosts/{{ inventory_hostname }}"
        method: DELETE
      when: cmdb_integration_enabled | default(false)
      tags: [decommission]

    - name: Securely wipe sensitive data directories (policy decision required)
      ansible.builtin.shell: shred -uz -n 3 {{ item }}
      loop: "{{ secure_wipe_paths | default([]) }}"
      when:
        - secure_wipe_enabled | default(false)
        - secure_wipe_paths is defined
      tags: [decommission]

    - name: Confirm decommission checklist complete
      ansible.builtin.debug:
        msg: "Host {{ inventory_hostname }} decommissioned. Manual step remaining: power off / reclaim hardware."
      tags: [decommission]
```

> **Note surfaced by building row-by-row:** Task 4 (revoke) and Task 5 (deregister monitoring) both act on things Task 6 (CMDB) and Task 8 (final debug message) reference — order here follows the table's row order, which happens to already be the correct dependency order (stop → notify externals → back up → revoke credentials → deregister monitoring → deregister CMDB → wipe → manual power-off). No reordering was needed this time, unlike the hold-package ordering issue found in the patch playbook build.

---

## Next steps

- Once both playbooks are validated (commissioning against a test VM, decommissioning read-through only), fill in the real file paths for `import_playbook: hardening.yml` and `import_playbook: patch_management.yml` from your actual prior-session playbooks.
- Decide and document the `secure_wipe_enabled`/`secure_wipe_paths` policy (§4.4 Task 7) before this is needed for real — that's explicitly flagged as a decision, not a default.
- Carry the CMDB-schema note (two-state vs. single-state) and the exercise findings into the Gaps & Action Items template (§8 of the workshop doc).
