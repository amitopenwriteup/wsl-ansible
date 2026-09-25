# Step-by-Step Process: Ubuntu Configuration Baseline & Autopatching (Local Testing, No Artifact Server)

This turns the attached workshop into a straight step-by-step build process — no exercises, no fill-in-the-blanks, just the order to create things and the commands to run. It keeps the change from your last message: a `local` group so you can test everything against the control node itself (`ansible_connection=local`) before pointing it at real Ubuntu hosts, and no artifact server — the login banner is rendered from a local template and patching pulls only from each host's default `apt` sources.

---

## Step 1 — Confirm prerequisites

| Item | Details |
|---|---|
| Control node | Linux/macOS/WSL with Ansible 2.14+ (`ansible --version`) |
| Target | The control node itself (`local` group) to start; real Ubuntu 22.04+ hosts once you're ready (`ubuntu` group) |
| Access | If using real hosts: SSH key login, user with passwordless `sudo` |

---

## Step 2 — Create the project structure

```bash
mkdir ansible-workshop && cd ansible-workshop
mkdir -p templates reports
touch ansible.cfg inventory.ini vars.yml baseline.yml patch.yml site.yml
```

End state:

```text
ansible-workshop/
├── ansible.cfg
├── inventory.ini
├── vars.yml
├── baseline.yml
├── patch.yml
├── site.yml
├── templates/
│   ├── issue.net.j2
│   └── 99-baseline.conf.j2
└── reports/
```

---

## Step 3 — Write `ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
interpreter_python = auto_silent
retry_files_enabled = False
```

---

## Step 4 — Write `inventory.ini`

Two groups: `local` (the control node itself, for fast testing with no SSH involved) and `ubuntu` (real hosts, add them when you have them).

```ini
[local]
localhost ansible_connection=local

[ubuntu]
# ubu-01 ansible_host=192.168.56.31
# add real hosts here when ready, one per line

[ubuntu:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
```

**Test connectivity against the local group first:**

```bash
ansible local -m ansible.builtin.ping
```

- [ ] Returns `pong`

When you add real hosts under `[ubuntu]`, test those too: `ansible ubuntu -m ansible.builtin.ping`.

---

## Step 5 — Write `vars.yml`

One file, no artifact-server variables, everything both playbooks need:

```yaml
# ---- configuration baseline ----
baseline_packages:
  - chrony
  - curl
  - cron
  - ca-certificates
baseline_absent_packages:
  - telnet
baseline_ssh_permit_root_login: "no"
baseline_ssh_max_auth_tries: 3

# ---- patching defaults ----
patch_min_free_mb: 2048
patch_hold_packages: []
patch_allow_reboot: false
patch_critical_services:
  - ssh.service
  - cron.service
```

Override anything for a single run with `-e` — it always beats `vars.yml`.

---

## Step 6 — Write `templates/issue.net.j2`

No artifact server: the banner is rendered locally, not downloaded.

```jinja
Authorized access only. Activity may be monitored.
```

---

## Step 7 — Write `templates/99-baseline.conf.j2`

The SSH policy drop-in, rendered from variables:

```jinja
# {{ ansible_managed }}
PermitRootLogin {{ baseline_ssh_permit_root_login }}
MaxAuthTries {{ baseline_ssh_max_auth_tries }}
Banner /etc/issue.net
```

---

## Step 8 — Write `baseline.yml`

```yaml
---
- name: Enforce Ubuntu configuration baseline
  hosts: "{{ target | default('local') }}"
  become: true
  gather_facts: true
  vars_files:
    - vars.yml

  tasks:
    - name: Assert host is Ubuntu 22.04 or newer
      ansible.builtin.assert:
        that:
          - ansible_facts['distribution'] == 'Ubuntu'
          - ansible_facts['distribution_version'] is version('22.04', '>=')
        fail_msg: "Unsupported OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
        success_msg: "OS check passed"

    - name: Ensure required packages are installed
      ansible.builtin.apt:
        name: "{{ baseline_packages }}"
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Ensure forbidden packages are removed
      ansible.builtin.apt:
        name: "{{ baseline_absent_packages }}"
        state: absent

    - name: Deploy the login banner
      ansible.builtin.template:
        src: issue.net.j2
        dest: /etc/issue.net
        owner: root
        group: root
        mode: "0644"

    - name: Deploy SSH baseline drop-in
      ansible.builtin.template:
        src: 99-baseline.conf.j2
        dest: /etc/ssh/sshd_config.d/99-baseline.conf
        owner: root
        group: root
        mode: "0644"
      notify:
        - Validate sshd config
        - Restart ssh

    - name: Enforce permissions on /etc/crontab
      ansible.builtin.file:
        path: /etc/crontab
        owner: root
        group: root
        mode: "0644"

    - name: Ensure cron is running and enabled
      ansible.builtin.service:
        name: cron
        state: started
        enabled: true

  handlers:
    - name: Validate sshd config
      ansible.builtin.command: sshd -t
      changed_when: false

    - name: Restart ssh
      ansible.builtin.service:
        name: ssh
        state: restarted
```

**Notice `hosts:` defaults to `local`, not `ubuntu`.** That's deliberate — the safe default is the control node you're sitting at, not a fleet. Once you trust the playbook, run it against real hosts explicitly with `-e target=ubuntu`.

**Run it:**

```bash
ansible-playbook baseline.yml --syntax-check
ansible-playbook baseline.yml            # runs against 'local' by default
ansible-playbook baseline.yml            # run again — expect changed=0
```

- [ ] Syntax check passes
- [ ] Second run shows `changed=0`

---

## Step 9 — Write `patch.yml`

This is the autopatching playbook. It never needed an artifact server — `apt: upgrade: dist` already pulls only from whatever's configured in `/etc/apt/sources.list` on the target (Ubuntu's default `archive.ubuntu.com` / `security.ubuntu.com` mirrors, normally).

```yaml
---
- name: Patch Ubuntu servers
  hosts: "{{ target | default('local') }}"
  become: true
  serial: "{{ patch_serial | default(1) }}"
  any_errors_fatal: true
  vars_files:
    - vars.yml

  tasks:
    # ---------- PRE-CHECKS ----------
    - name: Gather package facts
      ansible.builtin.package_facts:
        manager: auto
      tags: [precheck]

    - name: Assert enough free disk space on /
      ansible.builtin.assert:
        that:
          - >-
            (ansible_facts['mounts']
             | selectattr('mount', 'equalto', '/')
             | map(attribute='size_available')
             | first) > (patch_min_free_mb | int * 1024 * 1024)
        fail_msg: "Not enough free space on / (need {{ patch_min_free_mb }} MB)"
      tags: [precheck]

    - name: Refresh apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
        lock_timeout: 120
      tags: [precheck]

    - name: List pending updates (read-only)
      ansible.builtin.command: apt list --upgradable
      register: pending_updates
      changed_when: false
      check_mode: false
      tags: [precheck]

    - name: Set pending update count
      ansible.builtin.set_fact:
        pending_count: "{{ pending_updates.stdout_lines | select('search', 'upgradable from') | list | length }}"
      tags: [precheck]

    - name: Show pending update count
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: {{ pending_count }} updates pending"
      tags: [precheck]

    - name: Record running kernel before patching
      ansible.builtin.command: uname -r
      register: kernel_before
      changed_when: false
      check_mode: false
      tags: [precheck]

    # ---------- PATCH ----------
    - name: Patch workflow
      tags: [patch]
      block:
        - name: Hold pinned packages
          ansible.builtin.dpkg_selections:
            name: "{{ item }}"
            selection: hold
          loop: "{{ patch_hold_packages }}"
          when: item in ansible_facts['packages']

        - name: Apply all available updates
          ansible.builtin.apt:
            upgrade: dist
            autoremove: true
            autoclean: true
            lock_timeout: 120
          register: patch_result

        - name: Check whether a reboot is required
          ansible.builtin.stat:
            path: /var/run/reboot-required
          register: reboot_flag

        - name: Reboot when required and permitted
          ansible.builtin.reboot:
            reboot_timeout: 900
            msg: "Reboot initiated by Ansible patching"
          when:
            - patch_allow_reboot | bool
            - reboot_flag.stat.exists
          register: reboot_result

        - name: Record running kernel after patching
          ansible.builtin.command: uname -r
          register: kernel_after
          changed_when: false
          check_mode: false

        - name: Gather service facts
          ansible.builtin.service_facts:

        - name: Assert critical services are running
          ansible.builtin.assert:
            that:
              - item in ansible_facts['services']
              - ansible_facts['services'][item]['state'] == 'running'
            fail_msg: "Critical service {{ item }} is not running after patching"
          loop: "{{ patch_critical_services }}"

      rescue:
        - name: Log the failure and stop this host
          ansible.builtin.fail:
            msg: "Patching failed on {{ inventory_hostname }}. See the task output above."

      always:
        - name: Ensure report directory exists
          ansible.builtin.file:
            path: /var/log/ansible-patching
            state: directory
            mode: "0750"

        - name: Write patch report on the host
          ansible.builtin.copy:
            dest: "/var/log/ansible-patching/patch-{{ ansible_facts['date_time']['iso8601_basic_short'] }}.log"
            mode: "0640"
            content: |
              host: {{ inventory_hostname }}
              time: {{ ansible_facts['date_time']['iso8601'] }}
              pending_before: {{ pending_count | default('n/a') }}
              packages_changed: {{ (patch_result | default({})).changed | default(false) }}
              reboot_required: {{ reboot_flag.stat.exists | default('unknown') }}
              rebooted: {{ (reboot_result | default({})) is changed }}
              kernel_before: {{ kernel_before.stdout | default('n/a') }}
              kernel_after: {{ kernel_after.stdout | default('n/a') }}
          register: report_file

        - name: Fetch report to the control node
          ansible.builtin.fetch:
            src: "{{ report_file.dest }}"
            dest: reports/
```

**Run it:**

```bash
ansible-playbook patch.yml --syntax-check
ansible-playbook patch.yml --tags precheck    # readiness report, changes nothing
ansible-playbook patch.yml --check --diff     # dry run
ansible-playbook patch.yml                    # patch 'local', no reboot
```

- [ ] Syntax check passes
- [ ] Precheck shows a pending-update count
- [ ] Patch run completes; a report lands in `reports/`

---

## Step 10 — Write `site.yml`

One entry point: baseline → patch → baseline, so you both fix drift and prove patching didn't reintroduce it.

```yaml
---
- import_playbook: baseline.yml
- import_playbook: patch.yml
- import_playbook: baseline.yml
```

```bash
ansible-playbook site.yml
```

---

## Step 11 — Prove idempotency and drift detection

```bash
# Idempotency: second run must show changed=0
ansible-playbook baseline.yml
ansible-playbook baseline.yml

# Cause drift on purpose (on the local machine, since target defaults to 'local')
sudo systemctl stop cron

# Detect it without changing anything
ansible-playbook baseline.yml --check --diff
```

- [ ] Second baseline run: `changed=0`
- [ ] After stopping cron, `--check --diff` reports the cron task would change, nothing else does

---

## Step 12 — Move from `local` to real Ubuntu hosts

Once you're confident in `local`:

1. Add real host lines under `[ubuntu]` in `inventory.ini` (Step 4).
2. Confirm connectivity: `ansible ubuntu -m ansible.builtin.ping`
3. Run everything the same way, just pointing `target` at `ubuntu`:

```bash
ansible-playbook baseline.yml -e target=ubuntu
ansible-playbook patch.yml -e target=ubuntu --tags precheck
ansible-playbook patch.yml -e target=ubuntu
ansible-playbook site.yml -e target=ubuntu
```

Nothing else changes — same playbooks, same `vars.yml`, just a different `target`.

---

## Step 13 — Roll out gradually on real hosts

```bash
# Patch one host at a time, no reboot (the defaults)
ansible-playbook patch.yml -e target=ubuntu

# Widen the batch once you trust the result
ansible-playbook patch.yml -e target=ubuntu -e patch_serial=2

# Reboot window, once patching is verified
ansible-playbook patch.yml -e target=ubuntu -e patch_serial=2 -e patch_allow_reboot=true

# Target one host while testing
ansible-playbook patch.yml -e target=ubu-01
```

---

## Reference — full commands, in order

```bash
mkdir ansible-workshop && cd ansible-workshop
mkdir -p templates reports
# ... create ansible.cfg, inventory.ini, vars.yml, templates/*, baseline.yml, patch.yml, site.yml ...

ansible local -m ansible.builtin.ping

ansible-playbook baseline.yml --syntax-check
ansible-playbook baseline.yml
ansible-playbook baseline.yml            # expect changed=0

ansible-playbook patch.yml --syntax-check
ansible-playbook patch.yml --tags precheck
ansible-playbook patch.yml --check --diff
ansible-playbook patch.yml

ansible-playbook site.yml

# when ready for real hosts:
ansible ubuntu -m ansible.builtin.ping
ansible-playbook site.yml -e target=ubuntu
```
