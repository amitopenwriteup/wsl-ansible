# Ansible Labs (Unique Files, vi, Individual Run Commands)

Each lab now lives in its **own playbook file**, is opened with **`vi`**, and has
its **own `ansible-playbook -i inventory.ini <file>`** command — no shared file,
no `--tags` needed.

> **Note:** `port` is a reserved Ansible variable name (it collides with
> `ansible_port`), so every example below uses `http_port` instead.

---

# Lab 1: Loops, Conditionals, Handlers & More

## 1. `loop` — Repeating a Task

### Lab 1 — Basic loop

```bash
vi lab1_basic_loop.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Install a list of packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - git
        - curl
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab1_basic_loop.yaml
```

---

### Lab 2 — Loop over a variable

```bash
vi lab2_loop_var.yaml
```

```yaml
- name: test
  hosts: all
  vars:
    packages:
      - nginx
      - git
      - curl
  tasks:
    - name: Install packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab2_loop_var.yaml
```

---

### Lab 3 — Loop over a list of dictionaries

```bash
vi lab3_loop_dicts.yaml
```

```yaml
- name: test
  hosts: all
  become: true
  tasks:
    - name: Ensure groups exist
      ansible.builtin.group:
        name: "{{ item.groups }}"
        state: present
      loop:
        - { name: "alice", groups: "sudo" }
        - { name: "bob", groups: "developers" }

    - name: Create users
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      loop:
        - { name: "alice", groups: "sudo" }
        - { name: "bob", groups: "developers" }
```
> `sudo` already exists as a default group on Debian/Ubuntu, but `developers`
> does not — `ansible.builtin.user` won't create a missing group for you, so
> the "Ensure groups exist" task creates it first (`state: present` is a no-op
> for `sudo` since it's already there).

**Run:**
```bash
ansible-playbook -i inventory.ini lab3_loop_dicts.yaml
```

---

### Lab 4 — Naming the loop variable (`loop_control`)

```bash
vi lab4_loop_control.yaml
```

```yaml
- name: test
  hosts: all
  become: true
  tasks:
    - name: Ensure groups exist
      ansible.builtin.group:
        name: "{{ user.groups }}"
        state: present
      loop:
        - { name: "alice", groups: "sudo" }
        - { name: "bob", groups: "developers" }
      loop_control:
        loop_var: user

    - name: Create users
      ansible.builtin.user:
        name: "{{ user.name }}"
        groups: "{{ user.groups }}"
      loop:
        - { name: "alice", groups: "sudo" }
        - { name: "bob", groups: "developers" }
      loop_control:
        loop_var: user
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab4_loop_control.yaml
```

---

### Lab 5 — Workshop exercise: create three directories

```bash
vi lab5_loop_dirs.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Create app directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        mode: "0755"
      loop:
        - /opt/app/logs
        - /opt/app/data
        - /opt/app/config
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab5_loop_dirs.yaml
```

---

## 2. `when` — Conditional Execution

### Lab 6 — Basic condition

```bash
vi lab6_when_basic.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Restart service only on Debian-family hosts
      ansible.builtin.service:
        name: nginx
        state: restarted
      when: ansible_facts['os_family'] == "Debian"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab6_when_basic.yaml
```

---

### Lab 7 — Combining conditions (implicit AND)

```bash
vi lab7_when_and.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Only run on CentOS 8+ web servers
      ansible.builtin.debug:
        msg: "Matched!"
      when:
        - ansible_facts['distribution'] == "CentOS"
        - ansible_facts['distribution_major_version'] | int >= 8
        - "'webservers' in group_names"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab7_when_and.yaml
```

---

### Lab 8 — OR condition

```bash
vi lab8_when_or.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Run on CentOS or RedHat
      ansible.builtin.debug:
        msg: "Matched CentOS or RedHat!"
      when: ansible_facts['distribution'] == "CentOS" or ansible_facts['distribution'] == "RedHat"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab8_when_or.yaml
```

---

### Lab 9 — `when` with a registered variable

```bash
vi lab9_when_registered.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Check if a file exists
      ansible.builtin.stat:
        path: /etc/myapp.conf
      register: config_file

    - name: Only run if the file exists
      ansible.builtin.debug:
        msg: "Config found!"
      when: config_file.stat.exists
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab9_when_registered.yaml
```

---

### Lab 10 — `when` combined with a loop

```bash
vi lab10_when_loop.yaml
```

```yaml
- name: test
  hosts: all
  vars:
    packages:
      - nginx
      - git
      - curl
  tasks:
    - name: Install only the packages not already listed
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
      when: item != "curl"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab10_when_loop.yaml
```

---

## 3. Handlers — Run Tasks Only on Change, Only Once

### Lab 11 — Handler fires once even with two notifying tasks

`ansible.builtin.copy`'s `src:` is read from the **controller** (the machine
running `ansible-playbook`), not the target — so these files must exist next
to `lab11_handlers.yaml` before you run it:

```bash
vi nginx.conf
```
```
# minimal nginx.conf for the lab
events {}
http {
    server {
        listen 80;
    }
}
```

```bash
vi site.conf
```
```
server {
    listen 8080;
    server_name lab11.local;
}
```

```bash
vi lab11_handlers.yaml
```

```yaml
- name: test
  hosts: all
  become: true
  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Copy nginx config
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

    - name: Copy site config
      ansible.builtin.copy:
        src: site.conf
        dest: /etc/nginx/sites-available/site.conf
      notify: Restart nginx
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab11_handlers.yaml --ask-become-pass
```

---

### Lab 12 — Forcing handlers to run early (`flush_handlers`)

Reuses the same `nginx.conf` from Lab 11 — create it first if you haven't:

```bash
vi nginx.conf
```
```
events {}
http {
    server {
        listen 80;
    }
}
```

```bash
vi lab12_flush_handlers.yaml
```

```yaml
- name: test
  hosts: all
  become: true
  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Copy nginx config
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

    - name: Flush handlers now instead of at the end of the play
      ansible.builtin.meta: flush_handlers

    - name: Continue with more tasks after handler already ran
      ansible.builtin.debug:
        msg: "Handler already fired above"
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab12_flush_handlers.yaml --ask-become-pass
```

---

### Lab 13 — `listen`: one notify, many handlers

`ansible.builtin.template` also reads its `src:` from the controller, so
create the Jinja2 template first:

```bash
vi nginx.conf.j2
```
```
events {}
http {
    server {
        listen {{ http_port | default(80) }};
    }
}
```

```bash
vi lab13_listen.yaml
```

```yaml
- name: test
  hosts: all
  become: true
  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Deploy new config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: "restart web stack"
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
      listen: "restart web stack"

    - name: Clear nginx cache
      ansible.builtin.file:
        path: /var/cache/nginx
        state: absent
      listen: "restart web stack"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab13_listen.yaml --ask-become-pass
```

---

### Lab 14 — Workshop exercise: Reload systemd handler

Create a minimal (fake, for the lab) systemd unit file first:

```bash
vi myapp.service
```
```
[Unit]
Description=My demo app

[Service]
ExecStart=/bin/true

[Install]
WantedBy=multi-user.target
```

```bash
vi lab14_reload_systemd.yaml
```

```yaml
- name: test
  hosts: all
  become: true
  tasks:
    - name: Copy app systemd service file
      ansible.builtin.copy:
        src: myapp.service
        dest: /etc/systemd/system/myapp.service
      notify: Reload systemd
  handlers:
    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: true
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab14_reload_systemd.yaml --ask-become-pass
```

---

## 4. Other Essentials

### Lab 15 — `register`: capture a task's result

```bash
vi lab15_register.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Check disk usage
      ansible.builtin.command: df -h /
      register: disk_usage
      changed_when: false

    - name: Show the result
      ansible.builtin.debug:
        var: disk_usage.stdout
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab15_register.yaml
```

---

### Lab 16 — `block` / `rescue` / `always`

This lab is self-contained: it first creates two demo scripts on the target
(`start.sh`, which deliberately fails, and `rollback.sh`, which succeeds) so
`block`/`rescue`/`always` has something real to react to.

```bash
vi lab16_block_rescue_always.yaml
```

```yaml
- name: test
  hosts: all
  become: true
  tasks:
    - name: Ensure /opt/app exists
      ansible.builtin.file:
        path: /opt/app
        state: directory
        mode: "0755"

    - name: Deploy a start script that deliberately fails (for the demo)
      ansible.builtin.copy:
        dest: /opt/app/start.sh
        mode: "0755"
        content: |
          #!/bin/bash
          echo "Attempting to start the app..."
          exit 1

    - name: Deploy a rollback script that succeeds (for the demo)
      ansible.builtin.copy:
        dest: /opt/app/rollback.sh
        mode: "0755"
        content: |
          #!/bin/bash
          echo "Rolling back changes..."
          exit 0

    - name: Attempt risky operation with error handling
      block:
        - name: Try to start the app
          ansible.builtin.command: /opt/app/start.sh
      rescue:
        - name: Roll back on failure
          ansible.builtin.command: /opt/app/rollback.sh
      always:
        - name: Always log the attempt
          ansible.builtin.debug:
            msg: "Deployment attempt finished"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab16_block_rescue_always.yaml --ask-become-pass
```

Expected recap: `start.sh` fails, `rescue` runs `rollback.sh` and succeeds, so
the play ends `ok` overall with `rescued=1` — no `failed` count, since the
rescue itself succeeded this time.

---

### Lab 17 — `tags`: run only part of a playbook

```bash
vi lab17_tags.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Install packages
      ansible.builtin.apt:
        name: nginx
        state: present
      tags: install

    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags: config
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab17_tags.yaml --tags install
ansible-playbook -i inventory.ini lab17_tags.yaml --skip-tags install
```

---

### Lab 18 — `vars` and `vars_files`

```bash
vi secrets_lab18.yml
```

```yaml
---
# put any secret values this playbook needs here
```

```bash
vi lab18_vars.yaml
```

```yaml
- name: test
  hosts: all
  vars:
    app_port: 8080
  vars_files:
    - secrets_lab18.yml
  tasks:
    - name: Show the app port
      ansible.builtin.debug:
        msg: "App runs on port {{ app_port }}"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab18_vars.yaml
```

---

### Lab 19 — Facts

```bash
vi lab19_facts.yaml
```

```yaml
- name: test
  hosts: all
  tasks:
    - name: Show the OS family
      ansible.builtin.debug:
        var: ansible_facts['os_family']
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab19_facts.yaml
```

---

## Quick Reference

| Lab | Topic | File |
|---|---|---|
| 1 | Basic loop | `lab1_basic_loop.yaml` |
| 2 | Loop over a variable | `lab2_loop_var.yaml` |
| 3 | Loop over dictionaries | `lab3_loop_dicts.yaml` |
| 4 | `loop_control` | `lab4_loop_control.yaml` |
| 5 | Loop exercise (directories) | `lab5_loop_dirs.yaml` |
| 6 | `when` basic | `lab6_when_basic.yaml` |
| 7 | `when` AND | `lab7_when_and.yaml` |
| 8 | `when` OR | `lab8_when_or.yaml` |
| 9 | `when` + registered var | `lab9_when_registered.yaml` |
| 10 | `when` + loop | `lab10_when_loop.yaml` |
| 11 | Handlers (notify once) | `lab11_handlers.yaml` |
| 12 | `flush_handlers` | `lab12_flush_handlers.yaml` |
| 13 | `listen` | `lab13_listen.yaml` |
| 14 | Reload systemd exercise | `lab14_reload_systemd.yaml` |
| 15 | `register` | `lab15_register.yaml` |
| 16 | `block/rescue/always` | `lab16_block_rescue_always.yaml` |
| 17 | `tags` | `lab17_tags.yaml` |
| 18 | `vars` + `vars_files` | `lab18_vars.yaml` |
| 19 | Facts | `lab19_facts.yaml` |
