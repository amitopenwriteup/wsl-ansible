# Workshop: Configuration Management & Patch Management on Ubuntu with Ansible



---

## 4. Skill: How to Research Modules on the Ansible Docs Site

Before writing any task, learn to find the right module yourself. This is the most reusable skill in the workshop.

### 4.1 Where to look

| Source | Best for | How |
|---|---|---|
| **Docs site, builtin index** | Browsing, reading examples | <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html> |
| **Docs site, one module** | Full reference | `https://docs.ansible.com/ansible/latest/collections/ansible/builtin/<module>_module.html` (example: `get_url_module.html`) |
| **`ansible-doc` (offline)** | Exact docs for the version you have installed | `ansible-doc ansible.builtin.get_url` |
| **`ansible-doc -l`** | Finding a module by keyword | `ansible-doc -l \| grep -i download` |
| **`ansible-doc -s`** | A copy-paste snippet of all options | `ansible-doc -s ansible.builtin.get_url` |
| **`ansible-doc -t keyword`** | Play/task keywords such as `serial` | `ansible-doc -t keyword serial` |

**Version tip:** the website shows the `latest` docs. Check your installed version with `ansible --version`. If they differ, trust `ansible-doc`, or switch the version in the docs site's selector.

### 4.2 How to read a module page (in this order)

1. **Synopsis** – does it do what you need?
2. **Parameters** – which are **required**? What are the defaults? Are there `choices`?
3. **Attributes** – does it support `check_mode` and `diff_mode`? (This decides whether it works for drift detection.)
4. **Notes / See Also** – gotchas and related modules
5. **Examples** – copy the closest one, then adapt it
6. **Return Values** – what you can `register` and use later

### 4.3 Decision flow: which module?

```text
Need to do X on a server
        │
        ▼
Is there a module for X?  ── search docs / ansible-doc -l
   │yes                        │no
   ▼                           ▼
Use it (FQCN)         Use ansible.builtin.command
                      + creates/removes or changed_when
                      Use shell only if you need pipes/redirects
```

**Why `ansible.builtin.` names?** `builtin` modules ship with `ansible-core`, so they exist on every installation. The fully qualified name removes ambiguity if another collection ships a module with the same short name.

### 4.4 Exercise: Module research worksheet (10 minutes)

Use the docs site or `ansible-doc` to fill in the **module** column. Then write one **parameter** you would set.

| Requirement | Module | Key parameter(s) |
|---|---|---|
| Download a file and verify its integrity | | |
| Install or remove packages on Ubuntu | | |
| Set owner and permissions on a file | | |
| Render a config file from a template | | |
| Make sure a service is running and enabled | | |
| Check whether a file exists | | |
| Stop the play if a condition is false | | |
| Reboot a host and wait for it to return | | |
| Prevent one package from being upgraded | | |
| Copy a file from the target back to the control node | | |

<details>
<summary>Click to reveal answers</summary>

| Requirement | Module | Key parameter(s) |
|---|---|---|
| Download a file and verify integrity | `ansible.builtin.get_url` | `url`, `dest`, `checksum` |
| Install or remove packages | `ansible.builtin.apt` | `name`, `state` |
| Owner and permissions | `ansible.builtin.file` | `path`, `owner`, `group`, `mode` |
| Render a template | `ansible.builtin.template` | `src`, `dest`, `mode` |
| Service running and enabled | `ansible.builtin.service` (or `systemd_service`) | `name`, `state`, `enabled` |
| Does a file exist? | `ansible.builtin.stat` | `path` (then use `.stat.exists`) |
| Stop if a condition is false | `ansible.builtin.assert` | `that`, `fail_msg` |
| Reboot and wait | `ansible.builtin.reboot` | `reboot_timeout` |
| Hold a package | `ansible.builtin.dpkg_selections` | `name`, `selection: hold` |
| Pull a file back | `ansible.builtin.fetch` | `src`, `dest` |

</details>

**Discussion:** for each requirement, what would the `command`/`shell` equivalent look like, and what would you lose (idempotency, check mode, diff, structured return values)?

- [ ] Worksheet completed

---

# Part A – Configuration Baseline and Drift

**Goal:** a playbook that describes the desired state of an Ubuntu server, fixes it when it drifts, and can report drift without changing anything.

**Desired state we will enforce**

| Area | Desired state |
|---|---|
| OS | Ubuntu 22.04 or newer |
| Packages | `chrony`, `curl`, `cron`, `ca-certificates` installed; `telnet` absent |
| Login banner | `/etc/issue.net` matches the approved file on the artifact server |
| SSH policy | Root login off, max 3 auth tries, banner enabled |
| File permissions | `/etc/crontab` is `root:root`, mode `0644` |
| Services | `cron` running and enabled |

---

### Step A1 – Create the play skeleton

**Requirement:** a play that targets the `ubuntu` group and loads the shared variables.

**Add to `baseline.yml`:**

```yaml
---
- name: Enforce Ubuntu configuration baseline
  hosts: "{{ target | default('ubuntu') }}"
  become: true
  gather_facts: true
  vars_files:
    - vars.yml

  tasks: []
```

**Why this way**

| Line | Reason |
|---|---|
| `hosts: "{{ target \| default('ubuntu') }}"` | Defaults to the whole inventory group, but `-e target=ubu-01` lets you target a single host while you're learning. |
| `vars_files: [vars.yml]` | Pulls in every variable from Section 3.4 without repeating it in the playbook. |
| `become: true` | Package, file, and service changes need root. |
| `gather_facts: true` | We need `ansible_facts` (OS name, version, mounts) in later tasks. |

**Run it**
```bash
ansible-playbook baseline.yml --syntax-check
ansible-playbook baseline.yml
```

- [ ] Syntax check passes and facts are gathered

---

### Step A2 – Guard: only run on supported Ubuntu

**Requirement:** stop early if the host is not Ubuntu 22.04+.

**Research:** find `assert` in the docs. Which parameters does it take? What do `fail_msg` and `success_msg` do?

**Predict:** if you run this against an Ubuntu 20.04 host, which state does the task end in, and does the play continue?

**Add under `tasks:`** (replace `tasks: []` with `tasks:`)

```yaml
  tasks:
    - name: Assert host is Ubuntu 22.04 or newer
      ansible.builtin.assert:
        that:
          - ansible_facts['distribution'] == 'Ubuntu'
          - ansible_facts['distribution_version'] is version('22.04', '>=')
        fail_msg: "Unsupported OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
        success_msg: "OS check passed"
```

**Why**

- A baseline written for Ubuntu 22.04+ (for example, the SSH drop-in directory) could misbehave on other systems. Failing **early and clearly** beats failing halfway through.
- `is version(...)` compares versions properly. String comparison would think `"9.10" > "22.04"`.

- [ ] Task returns `ok` on all hosts

---

### Step A3 – Enforce packages (**your turn**)

**Requirement:** required packages installed, forbidden packages removed.

**Research:** open `ansible.builtin.apt`. Find the values of `state` and the parameter that refreshes the package index.

**Fill in the blanks** (`____`):

```yaml
    - name: Ensure required packages are installed
      ansible.builtin.apt:
        name: "{{ baseline_packages }}"
        state: ____
        update_cache: true
        cache_valid_time: 3600

    - name: Ensure forbidden packages are removed
      ansible.builtin.apt:
        name: "{{ baseline_absent_packages }}"
        state: ____
```

<details>
<summary>Answer</summary>

`present` for the first task, `absent` for the second.

</details>

**Why**

| Choice | Reason |
|---|---|
| A **list** in one task | One `apt` transaction is faster than a loop with one package each |
| `cache_valid_time: 3600` | Skips `apt update` if the cache is under an hour old, so repeat runs stay fast and don't hammer mirrors |
| `state: present`, not `latest` | Baseline enforcement must not silently upgrade. Upgrades belong to the **patch** playbook (Part B). This separation is a key design idea. |
| `state: absent` | Drift includes things that should *not* be there |

**Predict, run twice.** First run: `changed` or `ok`? Second run?

- [ ] Second run shows `ok` for both tasks

---

### Step A4 – Download the approved baseline file with `get_url`

**Requirement:** every host's `/etc/issue.net` (login banner) must match a single approved file that lives on a central artifact server, and it must be verified.

#### A4.1 Prepare the artifact server (on the control node)

```bash
printf 'Authorized access only. Activity may be monitored.\n' > artifacts/issue.net
sha256sum artifacts/issue.net
cd artifacts && python3 -m http.server 8000
```

Leave the server running in a second terminal. Copy the SHA-256 hash into `vars.yml` as `issue_net_sha256`. If the control node has `ufw` enabled, allow the port: `sudo ufw allow 8000/tcp`.

Test from a target host:

```bash
curl -I http://192.168.56.1:8000/issue.net
```

> In production, serve artifacts over **HTTPS** from Artifactory, Nexus, S3, or an internal web server. `get_url` validates TLS certificates by default (`validate_certs: true`). Plain HTTP here is acceptable only because this is an isolated lab network.

#### A4.2 Research (this is the main "read the docs" exercise)

1. Open <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html> or run `ansible-doc ansible.builtin.get_url`.
2. Answer in your notes:
   - Which parameters are **required**?
   - What format does `checksum` expect?
   - What does `force` do, and what is its default?
   - What does `checksum` do when the file **already exists**?
   - Does the module support **check mode** and **diff mode**? (See *Attributes*.)
   - Which parameters would you use for a slow server or a private API needing a token? (`timeout`, `headers`)
   - Which return values can you `register` and print?

#### A4.3 Add the task (**your turn**)

```yaml
    - name: Download the approved login banner
      ansible.builtin.get_url:
        url: "http://{{ artifact_server }}:8000/issue.net"
        dest: /etc/issue.net
        checksum: "____:{{ issue_net_sha256 }}"
        owner: root
        group: root
        mode: "____"
```

<details>
<summary>Answer</summary>

`checksum: "sha256:{{ issue_net_sha256 }}"` and `mode: "0644"`.

</details>

#### A4.4 Why `get_url` and why these parameters

| Question | Answer |
|---|---|
| **Why not `command: curl -o`?** | Not idempotent (always "changed"), no integrity check, no diff, no check mode, no structured errors. |
| **Why not `copy`?** | `copy` pushes a file from the control node over SSH to every host. `get_url` makes each host **pull** from a central location, which scales better and keeps artifacts versioned in one place. Use `copy` when the file lives in your Git repo. |
| **Why `checksum`?** | Two jobs at once. (1) **Integrity:** the download fails if the hash doesn't match, so a tampered or truncated file is rejected. (2) **Drift enforcement:** if `dest` already exists and its hash matches, nothing is downloaded (`ok`). If someone edited the file, the hash differs, the file is re-downloaded, and the task reports `changed`. |
| **Why set `owner`, `group`, `mode` here?** | The file is *managed* in one place, including its permissions. |
| **When to use `force: true`?** | Rarely. It re-downloads every run and makes the task always `changed`. A checksum is better. |

**Predict**

1. First run: `changed` or `ok`?
2. Second run?
3. If you edit `/etc/issue.net` by hand and run again?

**Check the result on a host**

```bash
ansible ubuntu -b -m ansible.builtin.command -a "cat /etc/issue.net"
```

**Stretch:** register the result and print it.

```yaml
      register: banner_download

    - name: Show download result
      ansible.builtin.debug:
        var: banner_download
```

Look at `dest`, `checksum_dest`, `status_code`, and `msg`. Remove the debug task afterwards.

- [ ] First run `changed`, second run `ok`
- [ ] You can explain the difference between `get_url` and `copy`

---

### Step A5 – Manage SSH policy as a drop-in template

**Requirement:** enforce SSH settings without editing the vendor `sshd_config`.

**Create `templates/99-baseline.conf.j2`:**

```jinja
# {{ ansible_managed }}
PermitRootLogin {{ baseline_ssh_permit_root_login }}
MaxAuthTries {{ baseline_ssh_max_auth_tries }}
Banner /etc/issue.net
```

**Research:** open `ansible.builtin.template`. Which parameters set the source, destination, and permissions?

**Add to `baseline.yml`:**

```yaml
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
```

**Add handlers at the end of the play** (same indentation as `tasks:`):

```yaml
  handlers:
    - name: Validate sshd config
      ansible.builtin.command: sshd -t
      changed_when: false

    - name: Restart ssh
      ansible.builtin.service:
        name: ssh
        state: restarted
```

**Why**

| Choice | Reason |
|---|---|
| **Drop-in file** in `sshd_config.d/` | Ubuntu's `sshd_config` includes `sshd_config.d/*.conf`. Package upgrades can replace `sshd_config` but leave your drop-in alone. It is also easier to read, diff, and audit. |
| Filename starts with `99-` | sshd uses the **first** value it finds for a setting. Ubuntu's default file already sets some values, so read order matters. Check the include order if a setting seems ignored. |
| `template` | Values come from variables. `{{ ansible_managed }}` warns humans not to edit by hand. |
| **Handlers** | They run **only when the file changed**, and only once at the end of the play. Idempotent runs never restart SSH. |
| **Validate before restart** | Handlers run in the order they are **defined**. If `sshd -t` fails, the host fails and `Restart ssh` never runs, so you do not lock yourself out with a broken config. |
| `changed_when: false` | `sshd -t` only reads. It should never report "changed". |
| Service name `ssh` | On Ubuntu the service is `ssh`, not `sshd` (that is the RHEL name). |

**Predict:** which tasks/handlers run on the first run? On the second?

- [ ] Handlers fire on the first run only

---

### Step A6 – Enforce file permissions (**your turn**)

**Requirement:** `/etc/crontab` must be owned by `root:root` with mode `0644`.

**Research:** in `ansible.builtin.file`, what does `state: file` do differently from `state: touch`?

```yaml
    - name: Enforce permissions on /etc/crontab
      ansible.builtin.file:
        path: /etc/crontab
        owner: ____
        group: ____
        mode: "____"
```

<details>
<summary>Answer</summary>

`owner: root`, `group: root`, `mode: "0644"`.

</details>

**Why:** permission drift is common and dangerous, and `file` fixes it without touching the content. Quote `mode` as a string (`"0644"`), otherwise YAML may read `0644` as an octal number and confuse it.

- [ ] `ok` on the second run

---

### Step A7 – Enforce service state (**your turn**)

**Requirement:** `cron` must be running now and enabled at boot.

```yaml
    - name: Ensure cron is running and enabled
      ansible.builtin.service:
        name: cron
        state: ____
        enabled: ____
```

<details>
<summary>Answer</summary>

`state: started`, `enabled: true`.

</details>

**Why:** "running now" and "starts at boot" are two separate facts. Drift often hides in the difference.

- [ ] `ok` on the second run

---

### Step A8 – Prove idempotency

```bash
ansible-playbook baseline.yml
ansible-playbook baseline.yml
```

**Expected:** second run ends with `changed=0`. If not, find which task is not idempotent and fix it before moving on. Non-idempotent tasks make drift reports meaningless, because every run "changes" something.

- [ ] Second run: `changed=0`

---

### Step A9 – Cause drift on purpose

SSH into a target host and break things:

```bash
echo "tampered" | sudo tee /etc/issue.net
sudo chmod 0666 /etc/crontab
echo "PermitRootLogin yes" | sudo tee /etc/ssh/sshd_config.d/99-baseline.conf
sudo systemctl stop cron
sudo apt install -y telnet
```

**Predict (write it down before running):** in a dry run, which tasks will report `changed`?

| Task | Predicted state |
|---|---|
| Assert OS | |
| Install required packages | |
| Remove forbidden packages | |
| Download login banner | |
| SSH drop-in | |
| `/etc/crontab` permissions | |
| Cron service | |

**Detect without changing anything:**

```bash
ansible-playbook baseline.yml --check --diff
```

<details>
<summary>Expected result</summary>

`changed`: remove forbidden packages, download login banner, SSH drop-in (with a visible diff), `/etc/crontab` permissions (diff shows old/new mode), cron service. `ok`: OS assert, required packages.

Whether the banner shows `changed` in check mode depends on the module's check-mode support. You looked up **Attributes** in A4.2, so confirm it there.

</details>

**Read the diff.** Find the line that shows `PermitRootLogin yes` being replaced. This is your evidence of *what* drifted.

- [ ] Drift detected in check mode with nothing changed on the host

---

### Step A10 – Remediate and report

**Remediate:**

```bash
ansible-playbook baseline.yml
```

**Verify:**

```bash
ansible-playbook baseline.yml            # expect changed=0
```

**Create a simple drift report you can run on a schedule:**

```bash
ansible-playbook baseline.yml --check --diff | tee reports/drift-$(date +%F).log
grep -E 'changed=[1-9]' reports/drift-$(date +%F).log && echo "DRIFT DETECTED" || echo "NO DRIFT"
```

**How this works:** the PLAY RECAP line for each host shows `changed=N`. In check mode, `changed=N` means "N tasks *would* change", which is drift. `ansible-playbook` still exits with `0` in check mode, so you check the recap.

**Check-mode caveats (discuss)**

- `command` and `shell` tasks are **skipped** in check mode unless you set `check_mode: false` (only do that for read-only commands).
- A task that depends on the result of a skipped task may behave differently.
- Not every module supports check mode. Read **Attributes** in the docs.

**Choose an enforcement pattern (discuss with your partner):** which pattern from Section 2.3 would you use here? What would change if this fleet were production? Why?

- [ ] Drift remediated, second run `changed=0`
- [ ] Drift report produced

---

# Part B – Patch Management Playbook

**Goal:** a playbook that patches Ubuntu servers safely, in small batches, with pre-checks, holds, controlled reboots, post-checks, and a report per host.

**Design in one picture**

```text
choose batch size (-e patch_serial=...)
             │
             ▼
   ┌── for each batch of hosts ──────────────────────────────┐
   │ PRE-CHECKS  disk space, cache refresh, pending updates  │
   │ PATCH       holds → dist-upgrade → reboot? → post-check │
   │ REPORT      write log on host, fetch to control node    │
   └──────────────── stop the rollout on any failure ────────┘
```

Start a fresh `patch.yml` now.

---

### Step B1 – Play skeleton for a safe rollout

```yaml
---
- name: Patch Ubuntu servers
  hosts: "{{ target | default('ubuntu') }}"
  become: true
  serial: "{{ patch_serial | default(1) }}"
  any_errors_fatal: true
  vars_files:
    - vars.yml

  tasks: []
```

**Research:** `ansible-doc -t keyword serial` and `ansible-doc -t keyword any_errors_fatal`. What values does `serial` accept? (A number, a percentage, or a list of batch sizes.)

**Why**

| Line | Reason |
|---|---|
| `serial` default `1` | One host at a time. If patching breaks something, only one host is affected. Use `25%` or `2` for a bigger fleet. |
| `any_errors_fatal: true` | If one host fails, the whole run stops before the next batch. |
| `vars_files: [vars.yml]` | Same shared variables as `baseline.yml`, kept in one file. |
| `serial` set with `-e`, not in `vars.yml` | `serial` is a play-level setting, evaluated before host variables are applied, so it must come from extra vars or the play itself. |

**Run:** `ansible-playbook patch.yml --syntax-check`

- [ ] Syntax check passes

---

### Step B2 – Pre-check: gather package facts

**Requirement:** know what is installed before changing anything.

**Research:** `ansible-doc ansible.builtin.package_facts`. What does `manager: auto` do, and where do the results appear?

```yaml
  tasks:
    - name: Gather package facts
      ansible.builtin.package_facts:
        manager: auto
      tags: [precheck]
```

**Why:** later steps use `ansible_facts['packages']` to check if a package is installed (for holds). Tags let you run prechecks alone across the fleet.

- [ ] `ok` on all hosts

---

### Step B3 – Pre-check: enough free disk space

**Requirement:** fail the host early if `/` has less free space than `patch_min_free_mb`.

**Research:** run `ansible -m ansible.builtin.setup ubuntu -a "filter=ansible_mounts"`. Find `mount`, `size_available` (in **bytes**).

```yaml
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
```

**Why:** a package upgrade that runs out of disk space leaves dpkg half-configured, which is expensive to repair. Cheap checks first, expensive actions later.

**Break it:** run with `-e patch_min_free_mb=999999999`. Which state does the task end in and does the play continue?

- [ ] Task passes normally and fails when you raise the threshold

---

### Step B4 – Pre-check: refresh the cache and count pending updates

**Requirement:** refresh package metadata, then report how many updates are waiting, without installing anything.

**Research:** in `ansible.builtin.apt`, find `update_cache`, `cache_valid_time`, and `lock_timeout`.

```yaml
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
```

**Why**

| Choice | Reason |
|---|---|
| `lock_timeout: 120` | On Ubuntu, `unattended-upgrades` or another apt process may hold the lock. Waiting is better than failing. |
| `command` here | There is no builtin module that only *lists* upgradable packages, so this is the correct use of `command`. |
| `changed_when: false` | A read-only command must never say "changed". |
| `check_mode: false` | Otherwise the command would be skipped during `--check`, and `pending_updates` would be empty. |

**Run only the checks across the fleet:**

```bash
ansible-playbook patch.yml --tags precheck
```

That gives you a **patch readiness report** without touching anything. Useful the day before the window.

- [ ] Pending count printed for every host

---

### Step B5 – Record the kernel before patching

```yaml
    - name: Record running kernel before patching
      ansible.builtin.command: uname -r
      register: kernel_before
      changed_when: false
      check_mode: false
      tags: [precheck]
```

**Why:** a new kernel only takes effect after reboot. Comparing before and after gives **proof** the reboot applied it.

- [ ] Kernel version printed when you `debug` the variable

---

### Step B6 – Hold packages that must not move (**your turn**)

**Requirement:** packages listed in `patch_hold_packages` must never be upgraded by this playbook.

**Research:** `ansible-doc ansible.builtin.dpkg_selections`. Which values can `selection` take?

We will put the patch steps inside one `block:` so they share error handling later. Add this after the pre-checks:

```yaml
    - name: Patch workflow
      tags: [patch]
      block:
        - name: Hold pinned packages
          ansible.builtin.dpkg_selections:
            name: "{{ item }}"
            selection: ____
          loop: "{{ patch_hold_packages }}"
          when: item in ansible_facts['packages']
```

<details>
<summary>Answer</summary>

`selection: hold`

</details>

**Test it:** set `patch_hold_packages: [curl]` in `vars.yml` and re-run. Verify with `apt-mark showhold` on the host. Then set it back to `[]`, or add an "unhold" task as a stretch goal.

**Why**

- Databases, kernels of appliances, and vendor-certified packages often need change approval. A **hold** enforces that in code.
- `when: item in ansible_facts['packages']` avoids errors for packages that are not installed on a given host, and shows why the `package_facts` step (B2) exists.
- An empty list default means the loop does nothing. No special cases needed.

- [ ] Held package appears in `apt-mark showhold`

---

### Step B7 – Apply the updates

**Research:** in the `apt` docs, read the description of `upgrade`. What is the difference between `safe`, `full`, `yes`, and `dist`? Which one can install **new** packages or remove old ones to satisfy dependencies?

Add inside the `block:` after the hold task:

```yaml
        - name: Apply all available updates
          ansible.builtin.apt:
            upgrade: dist
            autoremove: true
            autoclean: true
            lock_timeout: 120
          register: patch_result
```

**Why**

| Choice | Reason |
|---|---|
| `upgrade: dist` | New kernel packages often arrive as *new* dependencies. A plain upgrade would hold them back. Confirm this in the docs description. |
| `autoremove` / `autoclean` | Clean up old kernels and cached `.deb` files so `/boot` and `/var` do not fill up over time. |
| `register: patch_result` | The result records whether anything changed, and the report will use it. |

**Dry run first:**

```bash
ansible-playbook patch.yml --check --diff
```

Then run for real:

```bash
ansible-playbook patch.yml
```

- [ ] Updates applied; second run shows `ok`

---

### Step B8 – Is a reboot required?

**Requirement:** detect whether the patches need a reboot.

**Research:** Ubuntu creates `/var/run/reboot-required` when a reboot is needed. Which module checks if a file exists, and how do you read the result?

```yaml
        - name: Check whether a reboot is required
          ansible.builtin.stat:
            path: /var/run/reboot-required
          register: reboot_flag
```

**Why `stat`:** it returns structured data (`reboot_flag.stat.exists`). No shell `test -f` needed, and it works in check mode.

- [ ] You can print `reboot_flag.stat.exists`

---

### Step B9 – Reboot only when required **and** allowed (**your turn**)

**Requirement:** reboot if the flag exists and reboots are currently allowed.

```yaml
        - name: Reboot when required and permitted
          ansible.builtin.reboot:
            reboot_timeout: 900
            msg: "Reboot initiated by Ansible patching"
          when:
            - ____ | bool
            - ____
          register: reboot_result
```

<details>
<summary>Answer</summary>

```yaml
          when:
            - patch_allow_reboot | bool
            - reboot_flag.stat.exists
```

</details>

**Why**

| Choice | Reason |
|---|---|
| `reboot` module | It waits for the host to go down, come back, and answer SSH before continuing. A manual `shell: reboot` with `wait_for` is fragile. |
| `patch_allow_reboot` | The **policy** lives in a variable, defaulting to `false` in `vars.yml`. |
| `reboot_timeout: 900` | Some servers take several minutes to boot (fsck, slow disks, hardware). |
| `| bool` | Variables passed with `-e` can arrive as strings. `bool` turns `"true"` into a real boolean. |

**Maintenance window pattern:** patch during the day *without* rebooting (the default). In the window, run:

```bash
ansible-playbook patch.yml -e patch_allow_reboot=true
```

`-e` beats `vars.yml`, so no file edit is needed.

- [ ] Host reboots only when `/var/run/reboot-required` exists

---

### Step B10 – Post-checks: prove the host is healthy

**Research:** `ansible-doc ansible.builtin.service_facts`. Facts appear under `ansible_facts['services']`, keyed by service name such as `ssh.service`.

```yaml
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
```

**Why:** "the upgrade command succeeded" does not mean "the server works". A post-check turns a silent failure into a stopped rollout, so the next batch never starts.

**Break it:** add a service that does not exist (for example `nosuch.service`) to `patch_critical_services` and observe the failure. Remove it afterwards.

- [ ] Assertion passes for `ssh.service` and `cron.service`

---

### Step B11 – Evidence: write and collect a report

Add **after** the `block:` content, as `rescue` and `always` sections of the same block. First the whole block skeleton, so indentation is clear:

```yaml
    - name: Patch workflow
      tags: [patch]
      block:
        # ... B6 – B10 tasks go here ...

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

**Research:** read the `block` section of the *Blocks* documentation on docs.ansible.com (search "Blocks"). When does `rescue` run? When does `always` run?

**Why**

| Choice | Reason |
|---|---|
| `block` / `rescue` / `always` | Like try / catch / finally. Tags on the block apply to every task inside. |
| `rescue` with `fail` | A `rescue` section **clears** the failure. Without `fail`, the host would be treated as success and the rollout would continue. We log and **re-raise**. |
| `always` | The report is written whether patching succeeded or failed, which is exactly when you need it. |
| `fetch` | Central evidence on the control node: `reports/<host>/var/log/ansible-patching/...`. |
| `default(...)` filters | If the block failed early, later variables don't exist yet, and the report must not crash. |

**Check the result:**

```bash
find reports -name 'patch-*.log' -exec cat {} \;
```

- [ ] A report exists for each patched host on the control node

---

### Step B12 – Roll out gradually

This is where the design pays off. **Same playbook, different variables, no separate files to edit.**

**1. Readiness check for the whole fleet (read-only):**
```bash
ansible-playbook patch.yml --tags precheck
```

**2. Patch one host at a time, no reboot (the defaults):**
```bash
ansible-playbook patch.yml
```

**3. Widen the batch once you trust the result:**
```bash
ansible-playbook patch.yml -e patch_serial=2
```

**4. Reboot window, once patching is verified:**
```bash
ansible-playbook patch.yml -e patch_serial=2 -e patch_allow_reboot=true
```

**5. Target a single host while testing:**
```bash
ansible-playbook patch.yml -e target=ubu-01
```

**Discussion questions**

1. What happens to the rollout if a host fails its post-check? Why?
2. How would you allow 10% of hosts to fail before stopping? (Hint: `max_fail_percentage`, see `ansible-doc -t keyword max_fail_percentage`)
3. What changes if you have 500 hosts? (Think `serial: [1, 5, "25%"]`, `forks`, and `throttle`.)
4. If you needed to patch some hosts on a different schedule than others (say, a small set of sensitive servers patched last, under change control), how would you extend this setup? (Hint: a second inventory group and a `-e target=...` override is the smallest change; a full split into per-group variable files is the next step up, and only worth it once the override list gets long.)
5. How does this map to your worksheet in Section 2.5?

**Stretch goals**

- [ ] Add an "unhold" task so packages removed from `patch_hold_packages` are released
- [ ] Add a maintenance-window guard: assert the host's local hour (`ansible_facts['date_time']['hour']`) is within an allowed range
- [ ] Add a `patch_ticket` variable required via `-e` and print it in the report
- [ ] Investigate limiting to security updates using the `default_release` parameter of `apt` (read the docs, then test it and compare the package list)
- [ ] Convert `patch.yml` into a role named `patching` (`defaults/`, `tasks/`, `templates/`, `meta/`), keeping `patch.yml` as a 10-line wrapper

- [ ] A full rollout completed with at least two different `patch_serial` values

---

# Part C – Bring It Together

### C1 – Does patching cause drift?

Patches can overwrite configuration. Prove your baseline survives.

```bash
ansible-playbook patch.yml
ansible-playbook baseline.yml --check --diff
```

**Predict:** `changed=0`? If not, which task detected drift, and what does that tell you about the upgrade?

### C2 – One entry point: `site.yml`

Create `site.yml`:

```yaml
---
- import_playbook: baseline.yml
- import_playbook: patch.yml
- import_playbook: baseline.yml
```

**Why:** enforce a clean baseline, patch, then verify the baseline again.

```bash
ansible-playbook site.yml
```

### C3 – Scheduling and automation

| Job | How |
|---|---|
| Nightly **drift report** | Cron/systemd timer on the control node running `baseline.yml --check --diff`, then alert on `changed=[1-9]` |
| Automatic **enforcement** | Scheduled run of `baseline.yml` without `--check` |
| **Patch window** | Scheduled job or manual approval that runs `patch.yml` |
| **Pull-based** enforcement | `ansible-pull -U <git-url> baseline.yml` on a timer on each host |
| **Enterprise** | Ansible Automation Platform or AWX: schedules, RBAC, credentials, and **surveys** to ask for `patch_serial`, `patch_ticket` |

Example cron entry (note the escaped `%`):

```cron
0 2 * * * cd /opt/ansible-workshop && ansible-playbook baseline.yml --check --diff >> /var/log/drift/$(date +\%F).log 2>&1
```

### C4 – Manage `unattended-upgrades` as code (bonus)

Existing automation often includes `unattended-upgrades`. Use what you learned:

- Ensure the package is installed (`apt`)
- Deploy `/etc/apt/apt.conf.d/20auto-upgrades` with a `template`
- Add it to `baseline.yml` so drift is detected
- Decide: should it coexist with your patch runs, or be disabled so patching stays under your control? Write your recommendation in two sentences.

---

## 8. Knowledge Check

1. Why is `state: present` used in the baseline but `upgrade: dist` in the patch playbook?
2. What are the two jobs of `checksum` in `get_url`?
3. Why does `serial` use an `-e` variable rather than living in `vars.yml`?
4. What does a `rescue` section do to the failed state of a host, and why do we call `fail` inside it?
5. Why are handlers ordered "validate, then restart"?
6. Which flag detects drift without changing anything? Which shows exactly what would change?
7. Why does `check_mode: false` appear on read-only `command` tasks?
8. Your fleet is patched during the day but must reboot at night. How do you do that with the same playbook?

<details>
<summary>Answers</summary>

1. The baseline must keep the system at a **defined** state without unplanned upgrades. Upgrades are a deliberate, controlled action in the patch playbook.
2. Integrity verification of the download, and drift enforcement (re-download only when the local file's hash differs).
3. It is a play-level setting, resolved before host variables are applied, so it must come from extra vars or the play.
4. `rescue` clears the failure, so the host would look successful. Calling `fail` re-raises it so `any_errors_fatal` can stop the rollout.
5. If the config is invalid, validation fails and restart never runs, so SSH is not restarted with a broken config.
6. `--check` and `--diff`.
7. Commands are skipped in check mode. Read-only commands whose output later tasks depend on must still run.
8. Leave `patch_allow_reboot: false` (the default) for the day run, then run again in the window with `-e patch_allow_reboot=true`.

</details>

---

## 9. Facilitator Notes

**Preparation**

- Provision 1–3 Ubuntu VMs per participant (or a shared pool per team) and test SSH and `sudo`.
- Start one shared artifact server for the whole room, or have each participant run their own on the control node.
- Keep a solution repository with the finished `baseline.yml`, `patch.yml`, and templates, released after each Part.

**Pacing tips**

- Enforce **Predict** before **Run**. Ask two or three people to share predictions each time.
- A4 (`get_url`) and B7 (`upgrade: dist`) are the two "read the docs" moments. Do not give the answer. Point to the *Parameters* and *Attributes* sections.
- If a group finishes early, hand out the stretch goals and C4.

**Common problems**

| Symptom | Cause / Fix |
|---|---|
| `Failed to lock apt` | Another apt process (often `unattended-upgrades`). `lock_timeout` helps; otherwise wait. |
| `get_url` cannot connect | Firewall on the control node blocks 8000. Open it or use the shared server. |
| Checksum mismatch | The artifact changed after you copied the hash. Recompute with `sha256sum`. |
| SSH drop-in ignored | An earlier file sets the same option first. Inspect `sshd -T` output and file order. |
| Reboot never returns | Increase `reboot_timeout`; confirm the VM's console. |
| `changed` on every run | A non-idempotent task, often `command` without `changed_when`, or `get_url` with `force: true`. |
| Variable ignored | Precedence. Check `-e` against `vars_files`. |

---

# Appendix A – Reference Solution: `baseline.yml`

```yaml
---
- name: Enforce Ubuntu configuration baseline
  hosts: "{{ target | default('ubuntu') }}"
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

    - name: Download the approved login banner
      ansible.builtin.get_url:
        url: "http://{{ artifact_server }}:8000/issue.net"
        dest: /etc/issue.net
        checksum: "sha256:{{ issue_net_sha256 }}"
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

# Appendix B – Reference Solution: `patch.yml`

```yaml
---
- name: Patch Ubuntu servers
  hosts: "{{ target | default('ubuntu') }}"
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

# Appendix C – Cheat Sheet

```bash
# --- Docs ---
ansible-doc -l | grep -i <keyword>              # find a module
ansible-doc ansible.builtin.get_url             # full docs
ansible-doc -s ansible.builtin.apt              # snippet with all options
ansible-doc -t keyword serial                   # play/task keyword docs

# --- Baseline / drift ---
ansible-playbook baseline.yml
ansible-playbook baseline.yml --check --diff

# --- Patching ---
ansible-playbook patch.yml --tags precheck
ansible-playbook patch.yml --check --diff
ansible-playbook patch.yml -e patch_serial=2
ansible-playbook patch.yml -e patch_allow_reboot=true

# --- Debugging ---
ansible-playbook patch.yml --syntax-check
ansible-playbook patch.yml --list-tasks
ansible-playbook patch.yml -vv
ansible-inventory --graph
```

| Module | Purpose in this workshop |
|---|---|
| `ansible.builtin.assert` | Guards and post-checks |
| `ansible.builtin.apt` | Packages, cache, upgrades |
| `ansible.builtin.get_url` | Download and verify baseline artifacts |
| `ansible.builtin.template` | Render config from variables |
| `ansible.builtin.file` | Ownership, permissions, directories |
| `ansible.builtin.service` | Service state and boot enablement |
| `ansible.builtin.package_facts` / `service_facts` | Inspect current state |
| `ansible.builtin.stat` | Reboot-required flag |
| `ansible.builtin.dpkg_selections` | Package holds |
| `ansible.builtin.reboot` | Controlled reboot |
| `ansible.builtin.copy` / `fetch` | Write and collect reports |
| `ansible.builtin.command` | Read-only checks with no module equivalent |

# Appendix D – Reference Links

- Builtin module index: <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html>
- `get_url`: <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html>
- `apt`: <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html>
- `reboot`: <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/reboot_module.html>
- `dpkg_selections`: <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dpkg_selections_module.html>
- Playbook guide (blocks, handlers, check mode, variables): <https://docs.ansible.com/ansible/latest/playbook_guide/index.html>
- Variable precedence: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html>
