# Workshop: Building an Ansible Role Step by Step

Turn a single playbook (nginx install, config copy, handler with `flush_handlers`) into a reusable role with `defaults`, `vars`, `handlers`, and `tasks/main.yml`.

**Time:** ~60-90 minutes | **Level:** Beginner to Intermediate

---

## Learning Objectives

By the end you will be able to:

- Add variables to an existing playbook
- Scaffold a role with `ansible-galaxy`
- Split logic into `defaults/`, `vars/`, `tasks/`, `handlers/`, `files/`
- Explain when to use `defaults` vs `vars`
- Use `meta: flush_handlers` inside a role
- Call the role from a playbook and override variables

## Prerequisites

- Ansible 2.14+ installed on the control node (`ansible --version`)
- A Debian/Ubuntu target host reachable over SSH (uses `apt`)
- `sudo` rights on the target
- An inventory file (example below)

```ini
# inventory.ini
[web]
192.168.1.10 ansible_user=ubuntu
```

Test connectivity:

```bash
ansible all -i inventory.ini -m ping
```

---
### vi Quick Reference

We create every file with `vi`. Learn these keys once:

| Key | What it does |
|---|---|
| `i` | Enter insert mode (start typing/pasting) |
| `Esc` | Leave insert mode |
| `:wq` + Enter | Save and quit |
| `:q!` + Enter | Quit without saving |
| `gg` then `dG` | Delete the whole file content (Esc first) |
| `:set paste` + Enter | Run before pasting to stop vi from breaking YAML indentation |

**Standard flow for every file:** `vi <file>` → `:set paste` → `i` → paste → `Esc` → `:wq`

---

## Step 1: Add Variables to the Playbook

Take your existing playbook (nginx install, config copy, `notify`, `meta: flush_handlers`, handler) and add a `vars:` block, replacing hard-coded values with variables.

```bash
vi site.yml
```

Inside vi: press `gg` then `dG` to clear the old content, `i` to insert, paste the content below, then `Esc` and `:wq` to save.

```yaml
- name: test
  hosts: all
  become: true
  vars:
    nginx_package: nginx
    nginx_service_name: nginx
    nginx_conf_src: nginx.conf
    nginx_conf_dest: /etc/nginx/nginx.conf
    nginx_update_cache: true
    nginx_restart_state: restarted
  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: "{{ nginx_package }}"
        state: present
        update_cache: "{{ nginx_update_cache }}"

    - name: Copy nginx config
      ansible.builtin.copy:
        src: "{{ nginx_conf_src }}"
        dest: "{{ nginx_conf_dest }}"
        owner: root
        group: root
        mode: "0644"
      notify: Restart nginx

    - name: Flush handlers now instead of at the end of the play
      ansible.builtin.meta: flush_handlers

    - name: Continue with more tasks after handler already ran
      ansible.builtin.debug:
        msg: "Handler already fired above for {{ nginx_service_name }}"
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: "{{ nginx_service_name }}"
        state: "{{ nginx_restart_state }}"
```

**Rules of thumb**

- Always quote a value that *starts* with `{{ }}`.
- Prefix variables with the role/app name (`nginx_...`) to avoid collisions.
- Keep the handler **name** static (`Restart nginx`) so `notify` matches reliably.

> We will create `nginx.conf` inside the role in Step 2 and run everything after the role is built.

---

## Step 2: Scaffold the Role

```bash
mkdir -p roles
ansible-galaxy role init roles/nginx
```

Result:

```text
roles/nginx/
├── README.md
├── defaults/
│   └── main.yml      # low-priority, overridable variables
├── files/            # static files (copy module looks here)
├── handlers/
│   └── main.yml      # handlers
├── meta/
│   └── main.yml      # role metadata and dependencies
├── tasks/
│   └── main.yml      # main task list
├── templates/        # Jinja2 templates (.j2)
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml      # high-priority, internal variables
```

Remove folders you do not need (optional):

```bash
rm -rf roles/nginx/tests
```

Some `ansible-galaxy` versions do not create the `files/` folder, so create it first (safe to run even if it already exists), then create the config file the role will copy:

```bash
mkdir -p roles/nginx/files
vi roles/nginx/files/nginx.conf
```

Press `i`, paste the following, then `Esc` and `:wq`:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events { worker_connections 768; }

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Which variable goes where?

| File | Precedence | Use for | Example |
|---|---|---|---|
| `defaults/main.yml` | **Lowest**, easy to override | Settings the *user* is expected to change | config source, cache update, restart state |
| `vars/main.yml` | **High**, hard to override | Internal constants the role depends on | package name, service name, destination path |

---

## Step 3: `defaults/main.yml`

User-facing knobs with safe defaults.

```bash
vi roles/nginx/defaults/main.yml
```

Inside vi: press `gg` then `dG` to clear the old content, `i` to insert, paste the content below, then `Esc` and `:wq` to save.

```yaml
---
# Name of the config file inside the role's files/ directory
nginx_conf_src: nginx.conf

# Refresh the apt cache before installing
nginx_update_cache: true

# State to apply when the config changes: restarted | reloaded
nginx_restart_state: restarted

# Message printed after handlers have been flushed
nginx_debug_msg: "Handler already fired above"
```

Anyone using the role can override these from a playbook, inventory, `group_vars`, or `-e`.

---

## Step 4: `vars/main.yml`

Internal values that users should not normally touch.

```bash
vi roles/nginx/vars/main.yml
```

Inside vi: press `gg` then `dG` to clear the old content, `i` to insert, paste the content below, then `Esc` and `:wq` to save.

```yaml
---
nginx_package: nginx
nginx_service_name: nginx
nginx_conf_dest: /etc/nginx/nginx.conf
nginx_conf_owner: root
nginx_conf_group: root
nginx_conf_mode: "0644"
```

> **Tip:** if a value could differ per OS (for example `apache2` vs `httpd`), load an OS-specific file with `include_vars` and keep it in `vars/`.

---

## Step 5: `tasks/main.yml`

Move the tasks out of the playbook. Note there is **no** `hosts`, `become`, or `handlers` here, because those belong to the playbook and to the role's `handlers/` folder.

```bash
vi roles/nginx/tasks/main.yml
```

Inside vi: press `gg` then `dG` to clear the old content, `i` to insert, paste the content below, then `Esc` and `:wq` to save.

```yaml
---
- name: Ensure nginx is installed
  ansible.builtin.apt:
    name: "{{ nginx_package }}"
    state: present
    update_cache: "{{ nginx_update_cache }}"

- name: Copy nginx config
  ansible.builtin.copy:
    src: "{{ nginx_conf_src }}"
    dest: "{{ nginx_conf_dest }}"
    owner: "{{ nginx_conf_owner }}"
    group: "{{ nginx_conf_group }}"
    mode: "{{ nginx_conf_mode }}"
  notify: Restart nginx

- name: Flush handlers now instead of at the end of the play
  ansible.builtin.meta: flush_handlers

- name: Continue with more tasks after handler already ran
  ansible.builtin.debug:
    msg: "{{ nginx_debug_msg }} ({{ nginx_service_name }})"
```

**Notes**

- `copy` with a relative `src` automatically searches `roles/nginx/files/`.
- `meta: flush_handlers` works inside roles and runs all handlers queued so far in the play.
- Add `tags:` to tasks later if you want selective runs (`--tags config`).

---

## Step 6: `handlers/main.yml`

```bash
vi roles/nginx/handlers/main.yml
```

Inside vi: press `gg` then `dG` to clear the old content, `i` to insert, paste the content below, then `Esc` and `:wq` to save.

```yaml
---
- name: Restart nginx
  ansible.builtin.service:
    name: "{{ nginx_service_name }}"
    state: "{{ nginx_restart_state }}"
```

Handlers are a plain list of tasks, with no `handlers:` key at the top.

---

## Step 7: Role Metadata (`meta/main.yml`)

```bash
vi roles/nginx/meta/main.yml
```

Inside vi: press `gg` then `dG` to clear the old content, `i` to insert, paste the content below, then `Esc` and `:wq` to save.

```yaml
---
galaxy_info:
  author: your_name
  description: Installs and configures nginx
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions: [all]
    - name: Debian
      versions: [all]

dependencies: []
```

---

## Step 8: Rewrite the Playbook to Use the Role

Open the playbook again, clear it, and replace it with a tiny version:

```bash
vi site.yml
```

Inside vi: press `gg` then `dG` to clear the old content, `i` to insert, paste the content below, then `Esc` and `:wq` to save.

```yaml
- name: test
  hosts: all
  become: true
  roles:
    - role: nginx
```



Final project layout:

```text
.
├── inventory.ini
├── site.yml
└── roles/
    └── nginx/
        ├── defaults/main.yml
        ├── files/nginx.conf
        ├── handlers/main.yml
        ├── meta/main.yml
        ├── tasks/main.yml
        └── vars/main.yml
```

---

## Step 9: Run and Verify

```bash
# 1. Syntax check
ansible-playbook -i inventory.ini site.yml --syntax-check

# 2. Dry run
ansible-playbook -i inventory.ini site.yml --check --diff

# 3. Real run
ansible-playbook -i inventory.ini site.yml

# 4. Idempotence check: second run should show changed=0
ansible-playbook -i inventory.ini site.yml

# 5. Verify on the target
ansible all -i inventory.ini -b -a "systemctl status nginx --no-pager"
ansible all -i inventory.ini -b -a "nginx -t"
```

**Expected first run:**

```text
TASK [nginx : Ensure nginx is installed]     changed
TASK [nginx : Copy nginx config]             changed
RUNNING HANDLER [nginx : Restart nginx]      changed
TASK [nginx : Continue with more tasks ...]  ok
```

**Expected second run:** everything `ok`, no handler executed.

Optional linting:

```bash
pip install ansible-lint
ansible-lint roles/nginx
```

---

## Step 10: Prove You Understand Precedence

Override a variable at different levels and observe which wins.

```bash
# defaults value is overridden by -e (highest priority)
ansible-playbook -i inventory.ini site.yml -e "nginx_restart_state=reloaded"

# vars/main.yml value is NOT easily overridden by role params or inventory vars
ansible-playbook -i inventory.ini site.yml -e "nginx_package=nginx-light"
```

Simplified order (low to high):

1. `defaults/main.yml`
2. Inventory / `group_vars` / `host_vars`
3. Play `vars:`
4. `vars/main.yml` (role vars)
5. Role parameters and task `vars:`
6. Extra vars `-e` (always wins)

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `could not find or access 'nginx.conf'` | File not in `roles/nginx/files/` | Run `mkdir -p roles/nginx/files`, then `vi roles/nginx/files/nginx.conf` |
| Handler never runs | Task returned `ok`, not `changed`, or name mismatch | Match `notify` text exactly to handler `name` |
| `template error while templating string` | Unquoted `{{ }}` at start of a value | Wrap in quotes |
| Variable override ignored | It is defined in `vars/`, not `defaults/` | Move it to `defaults/` |
| `roles` folder not found | Running from the wrong directory | Run from the project root or set `roles_path` in `ansible.cfg` |
| Nginx fails after restart | Bad config file | Add `validate: "nginx -t -c %s"` to the `copy` task |

Safer config copy with validation:

```yaml
- name: Copy nginx config
  ansible.builtin.copy:
    src: "{{ nginx_conf_src }}"
    dest: "{{ nginx_conf_dest }}"
    validate: "nginx -t -c %s"
  notify: Restart nginx
```

---

## Hands-On Exercises

1. **Template it:** replace `copy` with `ansible.builtin.template` and create `templates/nginx.conf.j2` using a `nginx_worker_connections` variable (default `768`).
2. **Reload not restart:** set `nginx_restart_state: reloaded` from the playbook and confirm nginx keeps its PID.
3. **Enable on boot:** add a task using `ansible.builtin.service` with `enabled: true` and `state: started`.
4. **Add a firewall step:** open port 80 with `community.general.ufw`, controlled by a `nginx_open_firewall` boolean (default `false`).
5. **Tags:** tag tasks `install` and `config`, then run only `--tags config`.
6. **Second handler:** add a `Reload nginx` handler and trigger it from a vhost task.

---

## Cheat Sheet

```bash
ansible-galaxy role init roles/<name>        # scaffold
ansible-playbook site.yml --syntax-check     # syntax
ansible-playbook site.yml --check --diff     # dry run
ansible-playbook site.yml -e "var=value"     # override variable
ansible-playbook site.yml --tags config      # only tagged tasks
ansible-playbook site.yml --list-tasks       # show tasks
ansible-doc ansible.builtin.copy             # module docs
```

| Role folder | Purpose |
|---|---|
| `tasks/` | What to do |
| `handlers/` | React to changes |
| `defaults/` | Overridable settings |
| `vars/` | Internal constants |
| `files/` | Static files for `copy` |
| `templates/` | Jinja2 files for `template` |
| `meta/` | Metadata and dependencies |

---

## Summary

1. Started from your existing playbook.
2. Added variables to make it flexible.
3. Scaffolded a role with `ansible-galaxy role init`.
4. Split logic into `defaults`, `vars`, `tasks`, and `handlers`.
5. Called the role from a minimal playbook and verified idempotence.

You now have a reusable, overridable nginx role you can drop into any project.
