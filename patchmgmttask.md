# Ubuntu Patch Management Playbook — Incremental Build (Task-by-Task)

**Companion to:** *Workshop: Patch Management Playbook Design for Ubuntu*
**Purpose:** Build the playbook one task at a time, in the exact order the rows appear in §4.1 (Pre-flight), §4.2 (Backup), and §4.3 (Upgrade), so each addition can be reviewed and validated against the current automation before the next is appended.

---

## How to use this document

- Each task below corresponds to exactly one row of the module table in the referenced section.
- Tasks are appended in row order — do not skip ahead.
- After each task, re-run against the non-prod test group (`--tags <stage>`) and confirm before moving to the next.
- The final section (§5) shows the fully assembled playbook after all tasks from 4.1–4.3 have been appended.

---

## §4.1 Pre-flight — task-by-task

### Task 1 of 4 — Gather facts

| Row | Module |
|---|---|
| Gather facts / OS version, disk, memory | `ansible.builtin.setup` |

```yaml
---
- name: Ubuntu Patch Management Playbook
  hosts: patch_targets
  become: true
  serial: "{{ patch_batch_size | default('20%') }}"
  vars:
    upgrade_mode: safe
    min_free_space_mb: 2048
    reboot_allowed: true
    health_check_port: 443

  tasks:
    - name: Pre-flight - gather facts
      ansible.builtin.setup:
      tags: [preflight]
```

> **Check before continuing:** does the current repo already disable `gather_facts` at the play level? If so, this task is redundant and can be dropped instead of appended.

---

### Task 2 of 4 — Check disk space

| Row | Module |
|---|---|
| Check disk space before patching | `ansible.builtin.shell`/`ansible.builtin.command` with `df` — or assert against `ansible_facts['mounts']` |

```yaml
    - name: Pre-flight - capture available disk space on root
      ansible.builtin.set_fact:
        root_free_bytes: "{{ ansible_facts['mounts'] | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first }}"
      tags: [preflight]
```

> Uses the facts-based approach (preferred per the table's note) rather than raw `shell`/`df`, so the value is idempotent and reusable in Task 3.

---

### Task 3 of 4 — Assert minimum free space

| Row | Module |
|---|---|
| Assert minimum free space | `ansible.builtin.assert` |

```yaml
    - name: Pre-flight - assert sufficient disk space
      ansible.builtin.assert:
        that: root_free_bytes | int > (min_free_space_mb * 1024 * 1024)
        fail_msg: "Insufficient disk space to proceed with patching"
      tags: [preflight]
```

> This is the fail-fast gate — nothing after this point should run if disk space is insufficient.

---

### Task 4 of 4 — Check for existing pending reboot

| Row | Module |
|---|---|
| Check for existing pending reboot | `ansible.builtin.stat` on `/var/run/reboot-required` |

```yaml
    - name: Pre-flight - check for existing pending reboot
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: pre_existing_reboot_flag
      tags: [preflight]
```

> This captures whether a host was *already* pending reboot before this run started — useful for distinguishing "reboot caused by this patch cycle" from "reboot backlog from a prior cycle" in reporting later.

**§4.1 complete.** Validate with `--tags preflight` against the test group before moving to §4.2.

---

## §4.2 Backup — task-by-task

### Task 1 of 5 — Snapshot current package state (dpkg selections)

| Row | Module |
|---|---|
| Snapshot current package state (dpkg selections) | `ansible.builtin.shell` (`dpkg --get-selections`), output written via `copy`/`template` |

```yaml
    - name: Backup - capture dpkg selections
      ansible.builtin.shell: dpkg --get-selections
      register: dpkg_selections_output
      changed_when: false
      tags: [backup]

    - name: Backup - write dpkg selections to file
      ansible.builtin.copy:
        content: "{{ dpkg_selections_output.stdout }}"
        dest: "/var/backups/dpkg_selections_{{ ansible_date_time.iso8601_basic_short }}.txt"
      tags: [backup]
```

> No pure built-in exists for this per the table's note — `shell` is used deliberately here, marked `changed_when: false` to keep it idempotent-safe for repeat runs.

---

### Task 2 of 5 — Backup config files/directories

| Row | Module |
|---|---|
| Backup config files/directories | `ansible.builtin.archive` |

```yaml
    - name: Backup - archive key config directories
      ansible.builtin.archive:
        path: /etc
        dest: "/var/backups/pre_patch_etc_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
      tags: [backup]
```

---

### Task 3 of 5 — Copy backup off-host

| Row | Module |
|---|---|
| Copy backup off-host | `ansible.builtin.fetch` |

```yaml
    - name: Backup - fetch config archive to control node
      ansible.builtin.fetch:
        src: "/var/backups/pre_patch_etc_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
        dest: "backups/{{ inventory_hostname }}/"
        flat: false
      tags: [backup]
```

---

### Task 4 of 5 — APT cache backup (optional, for offline rollback)

| Row | Module |
|---|---|
| APT cache backup (optional, for offline rollback) | `ansible.builtin.copy` of `/var/cache/apt/archives` |

```yaml
    - name: Backup - preserve apt cache for offline rollback
      ansible.builtin.copy:
        src: /var/cache/apt/archives/
        dest: "/var/backups/apt_archives_{{ ansible_date_time.iso8601_basic_short }}/"
        remote_src: true
      when: rollback_strategy | default('none') == 'package_pin'
      tags: [backup]
```

> Marked optional per the table — gated here behind a `rollback_strategy` variable so it only runs for groups that have chosen package-version-pin rollback (see §6.5 of the workshop doc).

---

### Task 5 of 5 — Record pre-patch package versions for diffing

| Row | Module |
|---|---|
| Record pre-patch package versions for diffing | `ansible.builtin.package_facts` |

```yaml
    - name: Backup - capture pre-patch package facts
      ansible.builtin.package_facts:
      tags: [backup]

    - name: Backup - snapshot pre-patch package facts to a fact for later diff
      ansible.builtin.set_fact:
        pre_patch_packages: "{{ ansible_facts.packages }}"
      tags: [backup]
```

> `pre_patch_packages` is the baseline that §4.4 post-health "compare package versions" task will diff against later.

**§4.2 complete.** Validate with `--tags preflight,backup` (per the Section 7 exercise) before moving to §4.3.

---

## §4.3 Upgrade — task-by-task

### Task 1 of 7 — Update apt cache

| Row | Module |
|---|---|
| Update apt cache | `ansible.builtin.apt` with `update_cache: true` |

```yaml
    - name: Upgrade - refresh apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      tags: [upgrade]
```

---

### Task 2 of 7 — Security-only or full upgrade

| Row | Module |
|---|---|
| Security-only or full upgrade | `ansible.builtin.apt` with `upgrade: safe` or `upgrade: dist` |

```yaml
    - name: Upgrade - apply patches
      ansible.builtin.apt:
        upgrade: "{{ upgrade_mode }}"
      tags: [upgrade]
```

> `upgrade_mode` is externalized in `vars` (default `safe`) so the same task serves both security-only and full-`dist-upgrade` tiers, per the Scalability discussion (§6.4).

---

### Task 3 of 7 — Install/upgrade a specific package list

| Row | Module |
|---|---|
| Install/upgrade a specific package list | `ansible.builtin.apt` with `name: [...]`, `state: latest` |

```yaml
    - name: Upgrade - apply CVE-targeted package list (optional)
      ansible.builtin.apt:
        name: "{{ cve_targeted_packages }}"
        state: latest
      when: cve_targeted_packages is defined and (cve_targeted_packages | length > 0)
      tags: [upgrade]
```

> Gated on an optional variable so this task is a no-op unless a group specifically defines a CVE-targeted list (rather than a blanket upgrade).

---

### Task 4 of 7 — Autoremove obsolete packages

| Row | Module |
|---|---|
| Autoremove obsolete packages | `ansible.builtin.apt` with `autoremove: true` |

```yaml
    - name: Upgrade - remove obsolete packages
      ansible.builtin.apt:
        autoremove: true
      tags: [upgrade]
```

> Table note: run after upgrade, not before — placement here respects that ordering.

---

### Task 5 of 7 — Hold specific packages from upgrading

| Row | Module |
|---|---|
| Hold specific packages from upgrading | `ansible.builtin.dpkg_selections` with `selection: hold` |

```yaml
    - name: Upgrade - hold pinned packages from upgrade
      ansible.builtin.dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop: "{{ held_packages | default([]) }}"
      tags: [upgrade]
```

> **Note:** logically this should run *before* Task 2 (the actual upgrade), since a package can't be excluded after the upgrade already applied. Flag this ordering during review — appending strictly row-by-row surfaces this kind of sequencing gap, which is exactly the point of the exercise.

---

### Task 6 of 7 — Detect if reboot is now required

| Row | Module |
|---|---|
| Detect if reboot is now required | `ansible.builtin.stat` on `/var/run/reboot-required` |

```yaml
    - name: Upgrade - check reboot requirement post-upgrade
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required_file
      tags: [upgrade]
```

> This is the *post*-upgrade re-check, distinct from the pre-existing check captured in §4.1 Task 4 — comparing the two tells you whether this patch cycle itself introduced the reboot requirement.

---

### Task 7 of 7 — Reboot host safely (with wait)

| Row | Module |
|---|---|
| Reboot host safely (with wait) | `ansible.builtin.reboot` |

```yaml
    - name: Upgrade - reboot if required and allowed
      ansible.builtin.reboot:
        reboot_timeout: 600
      when: reboot_required_file.stat.exists and reboot_allowed
      tags: [upgrade]
```

**§4.3 complete.** Validate with `--tags upgrade` (upgrade_mode: safe) and confirm reboot detection behaves as expected, per the Section 7 exercise, before moving on to §4.4 (post-health) and §4.5 (reporting).

---

## §5. Assembled playbook after §4.1–§4.3

The full file after all tasks above have been appended in order (§4.4 post-health and §4.5 reporting deliberately left off — those are the next two stages to append the same way):

```yaml
---
- name: Ubuntu Patch Management Playbook
  hosts: patch_targets
  become: true
  serial: "{{ patch_batch_size | default('20%') }}"
  vars:
    upgrade_mode: safe
    min_free_space_mb: 2048
    reboot_allowed: true
    health_check_port: 443

  tasks:
    # --- §4.1 Pre-flight ---
    - name: Pre-flight - gather facts
      ansible.builtin.setup:
      tags: [preflight]

    - name: Pre-flight - capture available disk space on root
      ansible.builtin.set_fact:
        root_free_bytes: "{{ ansible_facts['mounts'] | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first }}"
      tags: [preflight]

    - name: Pre-flight - assert sufficient disk space
      ansible.builtin.assert:
        that: root_free_bytes | int > (min_free_space_mb * 1024 * 1024)
        fail_msg: "Insufficient disk space to proceed with patching"
      tags: [preflight]

    - name: Pre-flight - check for existing pending reboot
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: pre_existing_reboot_flag
      tags: [preflight]

    # --- §4.2 Backup ---
    - name: Backup - capture dpkg selections
      ansible.builtin.shell: dpkg --get-selections
      register: dpkg_selections_output
      changed_when: false
      tags: [backup]

    - name: Backup - write dpkg selections to file
      ansible.builtin.copy:
        content: "{{ dpkg_selections_output.stdout }}"
        dest: "/var/backups/dpkg_selections_{{ ansible_date_time.iso8601_basic_short }}.txt"
      tags: [backup]

    - name: Backup - archive key config directories
      ansible.builtin.archive:
        path: /etc
        dest: "/var/backups/pre_patch_etc_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
      tags: [backup]

    - name: Backup - fetch config archive to control node
      ansible.builtin.fetch:
        src: "/var/backups/pre_patch_etc_{{ ansible_date_time.iso8601_basic_short }}.tar.gz"
        dest: "backups/{{ inventory_hostname }}/"
        flat: false
      tags: [backup]

    - name: Backup - preserve apt cache for offline rollback
      ansible.builtin.copy:
        src: /var/cache/apt/archives/
        dest: "/var/backups/apt_archives_{{ ansible_date_time.iso8601_basic_short }}/"
        remote_src: true
      when: rollback_strategy | default('none') == 'package_pin'
      tags: [backup]

    - name: Backup - capture pre-patch package facts
      ansible.builtin.package_facts:
      tags: [backup]

    - name: Backup - snapshot pre-patch package facts to a fact for later diff
      ansible.builtin.set_fact:
        pre_patch_packages: "{{ ansible_facts.packages }}"
      tags: [backup]

    # --- §4.3 Upgrade ---
    - name: Upgrade - refresh apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      tags: [upgrade]

    - name: Upgrade - hold pinned packages from upgrade
      ansible.builtin.dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop: "{{ held_packages | default([]) }}"
      tags: [upgrade]

    - name: Upgrade - apply patches
      ansible.builtin.apt:
        upgrade: "{{ upgrade_mode }}"
      tags: [upgrade]

    - name: Upgrade - apply CVE-targeted package list (optional)
      ansible.builtin.apt:
        name: "{{ cve_targeted_packages }}"
        state: latest
      when: cve_targeted_packages is defined and (cve_targeted_packages | length > 0)
      tags: [upgrade]

    - name: Upgrade - remove obsolete packages
      ansible.builtin.apt:
        autoremove: true
      tags: [upgrade]

    - name: Upgrade - check reboot requirement post-upgrade
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required_file
      tags: [upgrade]

    - name: Upgrade - reboot if required and allowed
      ansible.builtin.reboot:
        reboot_timeout: 600
      when: reboot_required_file.stat.exists and reboot_allowed
      tags: [upgrade]
```

> **Note on task ordering above:** the "hold pinned packages" task has been moved ahead of "apply patches" in this assembled version (unlike its row position in §4.3), since holds must be set before the upgrade runs to take effect. This is flagged inline in Task 5 of 7 above as a discussion point for the group.

---

## Next steps

- Append §4.4 (Post-patch health check) the same way, row by row, mapped against your repo's current definition of "healthy" (§3.1 point 5).
- Append §4.5 (Reporting/audit trail) last.
- Carry forward the ordering note above into the Gaps & Action Items template (§8 of the workshop doc).
