# Assignment: Convert an Nginx Playbook into an Ansible Role

**Topic:** Ansible – Playbooks, Variables, Templates, Handlers, Roles
**Target OS:** Ubuntu 22.04 / 24.04
**Difficulty:** Beginner → Intermediate
**Estimated time:** 60–90 minutes

---

## 1. Objective

You are given a working **single-file playbook** that installs and configures Nginx on Ubuntu. It uses variables, a Jinja2 template for `nginx.conf`, and handlers that validate the config and check the service.

Your job is to **refactor this playbook into a reusable Ansible role** named `nginx`, without changing its behaviour.

By the end you should be able to:

- Explain how a playbook maps to a role's directory structure
- Separate defaults, tasks, templates, and handlers into the right places
- Use variables to make a role configurable
- Use handlers to validate `nginx.conf` and check the `nginx` service
- Call the role from a small playbook

---

## 2. Prerequisites

- Ansible 2.14+ installed on the control node
- One or more Ubuntu hosts reachable over SSH (a local VM or container is fine)
- A user with `sudo` on the target host
- Basic YAML knowledge

Sample inventory (`inventory.ini`):

```ini
[webservers]
web01 ansible_host=192.168.56.10 ansible_user=ubuntu
```

Connectivity check:

```bash
ansible -i inventory.ini webservers -m ping
```

---

## 3. Provided Material

### 3.1 The starting playbook – `nginx_playbook.yml`

```yaml
---
- name: Install and configure Nginx on Ubuntu
  hosts: webservers
  become: true

  vars:
    nginx_package: nginx
    nginx_service: nginx
    nginx_conf_path: /etc/nginx/nginx.conf
    nginx_port: 80
    nginx_server_name: localhost
    nginx_worker_processes: auto
    nginx_worker_connections: 1024
    nginx_web_root: /var/www/html

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install Nginx
      ansible.builtin.apt:
        name: "{{ nginx_package }}"
        state: present

    - name: Deploy nginx.conf from template
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: "{{ nginx_conf_path }}"
        owner: root
        group: root
        mode: "0644"
        backup: true
      notify:
        - Validate nginx config
        - Reload nginx

    - name: Ensure Nginx is enabled and running
      ansible.builtin.service:
        name: "{{ nginx_service }}"
        state: started
        enabled: true

  handlers:
    - name: Validate nginx config
      ansible.builtin.command: nginx -t -c {{ nginx_conf_path }}
      register: nginx_config_check
      changed_when: false

    - name: Reload nginx
      ansible.builtin.service:
        name: "{{ nginx_service }}"
        state: reloaded

    - name: Check nginx service status
      ansible.builtin.command: systemctl is-active {{ nginx_service }}
      register: nginx_status
      changed_when: false
      failed_when: nginx_status.stdout != "active"
```

### 3.2 The template – `templates/nginx.conf.j2`

```jinja
# {{ ansible_managed }}
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    server {
        listen {{ nginx_port }};
        server_name {{ nginx_server_name }};

        root {{ nginx_web_root }};
        index index.html index.htm;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

> Note: the `Check nginx service status` handler is defined but not yet notified by any task. Part of your job is to wire it up correctly (see Task 4).

---

## 4. Your Tasks

### Task 1 – Run the original playbook

1. Save the playbook and the template (place the template in a `templates/` folder next to the playbook).
2. Run it:
   ```bash
   ansible-playbook -i inventory.ini nginx_playbook.yml
   ```
3. Confirm Nginx is running: open `http://<host-ip>` or run `curl -I http://<host-ip>`.
4. Run it a **second time** and note which tasks report `changed` vs `ok`. Explain why in your report.

### Task 2 – Create the role skeleton

Generate the role structure:

```bash
mkdir roles && cd roles
ansible-galaxy role init nginx
```

Remove any directories you do not use (for example `tests/`, `vars/` if empty) or leave them with a short comment explaining why they are empty.

### Task 3 – Move each piece to the correct location

Fill in the table below in your report, then do the migration.

| Playbook section | Role location |
|---|---|
| `vars:` (values users may override) | `roles/nginx/defaults/main.yml` |
| `tasks:` | `roles/nginx/tasks/main.yml` |
| `handlers:` | `roles/nginx/handlers/main.yml` |
| `nginx.conf.j2` | `roles/nginx/templates/nginx.conf.j2` |
| Role description, license, platforms | `roles/nginx/meta/main.yml` |

Requirements:

- All variables must have defaults in `defaults/main.yml`
- Prefix every variable with `nginx_` (already done in the provided playbook, keep it that way)
- Fully-qualified module names (`ansible.builtin.*`) must be used
- No hard-coded values in tasks or the template; everything configurable must be a variable

### Task 4 – Handlers and the service check

In `handlers/main.yml` your role must contain three handlers, executed in this order when `nginx.conf` changes:

1. **Validate nginx config** – runs `nginx -t` against the path in `nginx_conf_path` and stops the play if the config is invalid
2. **Reload nginx** – reloads the service named in `nginx_service`
3. **Check nginx service status** – confirms the service is `active` after the reload

Hints:

- Handlers run in the order they are **defined**, not the order they are notified
- Use `notify` with a list, or use `listen` to group handlers under one topic such as `nginx config changed`
- Handlers only run when the notifying task reports `changed`
- To run pending handlers before the end of the play, use `ansible.builtin.meta: flush_handlers`

### Task 5 – Add a config-file check variable

Add a new variable `nginx_validate_config` (default `true`).

- When `true`, the validate handler runs
- When `false`, it is skipped (use `when:` on the handler)

Add another variable, `nginx_check_service` (default `true`), to control the service-status handler in the same way.

### Task 6 – Write the calling playbook

Create `site.yml` that uses the role instead of inline tasks:

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_server_name: demo.local
```

Run it:

```bash
ansible-playbook -i inventory.ini site.yml
```

### Task 7 – Prove it works

Run each check and save the output (screenshot or pasted text) in your report:

```bash
# 1. Syntax check
ansible-playbook -i inventory.ini site.yml --syntax-check

# 2. Dry run
ansible-playbook -i inventory.ini site.yml --check --diff

# 3. Real run
ansible-playbook -i inventory.ini site.yml

# 4. Idempotency: second run must show changed=0
ansible-playbook -i inventory.ini site.yml

# 5. Verify on the server
ansible -i inventory.ini webservers -m command -a "nginx -t" -b
ansible -i inventory.ini webservers -m command -a "systemctl is-active nginx" -b
curl -I http://<host-ip>:8080
```

### Task 8 – Break it on purpose

1. Edit `nginx.conf.j2` and introduce a syntax error (for example remove a `;`).
2. Run the playbook.
3. Record what happens: which handler fails, and does Nginx keep serving the old config?
4. Fix the template and re-run.

Write 3–5 sentences explaining why validating the config **before** reloading is important.

---

## 5. Expected Final Structure

```text
ansible-nginx-assignment/
├── inventory.ini
├── site.yml
└── roles/
    └── nginx/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── meta/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   └── nginx.conf.j2
        └── README.md
```

---

## 6. Deliverables

Submit a single folder or zip containing:

1. The complete `ansible-nginx-assignment/` project
2. `roles/nginx/README.md` documenting:
   - What the role does
   - Supported OS
   - Every variable, its default, and its purpose (table)
   - An example playbook
3. `REPORT.md` containing:
   - Answers to Task 1 (why tasks changed or not on the second run) and Task 8
   - The completed mapping table from Task 3
   - Output or screenshots from Task 7

---

## 7. Grading Rubric (100 points)

| Criteria | Points |
|---|---|
| Correct role structure, unused directories handled | 10 |
| All variables moved to `defaults/main.yml` with sensible values | 15 |
| Tasks migrated correctly using `ansible.builtin.*` modules | 15 |
| Template works and uses variables only (no hard-coded values) | 10 |
| Handlers: validate config, reload, service check in the right order | 20 |
| `nginx_validate_config` and `nginx_check_service` toggles work | 10 |
| Idempotency: second run shows `changed=0` | 10 |
| README and REPORT are clear and complete | 10 |

**Bonus (up to +10):**

- Add `ansible-lint` to your workflow and submit a clean run (+3)
- Add a Molecule or simple Docker-based test scenario (+4)
- Add support for a second variable-driven virtual host template (+3)

---

## 8. Common Mistakes to Avoid

- Putting handlers in `tasks/main.yml` instead of `handlers/main.yml`
- Notifying a handler with a name that does not exactly match its `name:`
- Forgetting `become: true` so `apt` and `service` fail with permission errors
- Using `vars/main.yml` for values users should override (use `defaults/`)
- Using `command` where a module exists; keep `command` only for `nginx -t` and `systemctl is-active`
- Forgetting `changed_when: false` on read-only commands, which breaks idempotency reporting

---

## 9. Useful References

- Ansible roles: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html>
- Handlers: <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html>
- Template module: <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html>
- Nginx config testing: `nginx -t`
