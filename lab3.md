# Ansible Labs (Standardized Format)

All examples below use the same playbook skeleton:

```yaml
- name: test
  hosts: all
  tasks:
    - name: <task name>
      ...
```

Every lab lives in a single file, **`labloop.yaml`**, and is run with:

```bash
ansible-playbook -i inventory.ini labloop.yaml --tags <lab-tag>
```

Run the whole file at once with:

```bash
ansible-playbook -i inventory.ini labloop.yaml
```

---

# Lab 1: Loops, Conditionals, Handlers & More

## 1. `loop` — Repeating a Task

### Lab 1 — Basic loop
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
      tags: lab1_basic_loop
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab1_basic_loop
```

### Lab 2 — Loop over a variable
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
      tags: lab2_loop_var
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab2_loop_var
```

### Lab 3 — Loop over a list of dictionaries
```yaml
- name: test
  hosts: all
  tasks:
    - name: Create users
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      loop:
        - { name: "alice", groups: "sudo" }
        - { name: "bob", groups: "developers" }
      tags: lab3_loop_dicts
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab3_loop_dicts
```

### Lab 4 — Naming the loop variable (`loop_control`)
```yaml
- name: test
  hosts: all
  tasks:
    - name: Create users
      ansible.builtin.user:
        name: "{{ user.name }}"
        groups: "{{ user.groups }}"
      loop:
        - { name: "alice", groups: "sudo" }
        - { name: "bob", groups: "developers" }
      loop_control:
        loop_var: user
      tags: lab4_loop_control
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab4_loop_control
```

### Lab 5 — Workshop exercise: create three directories
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
      tags: lab5_loop_dirs
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab5_loop_dirs
```

---

## 2. `when` — Conditional Execution

### Lab 6 — Basic condition
```yaml
- name: test
  hosts: all
  tasks:
    - name: Restart service only on Debian-family hosts
      ansible.builtin.service:
        name: nginx
        state: restarted
      when: ansible_facts['os_family'] == "Debian"
      tags: lab6_when_basic
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab6_when_basic
```

### Lab 7 — Combining conditions (implicit AND)
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
      tags: lab7_when_and
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab7_when_and
```

### Lab 8 — OR condition
```yaml
- name: test
  hosts: all
  tasks:
    - name: Run on CentOS or RedHat
      ansible.builtin.debug:
        msg: "Matched CentOS or RedHat!"
      when: ansible_facts['distribution'] == "CentOS" or ansible_facts['distribution'] == "RedHat"
      tags: lab8_when_or
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab8_when_or
```

### Lab 9 — `when` with a registered variable
```yaml
- name: test
  hosts: all
  tasks:
    - name: Check if a file exists
      ansible.builtin.stat:
        path: /etc/myapp.conf
      register: config_file
      tags: lab9_when_registered

    - name: Only run if the file exists
      ansible.builtin.debug:
        msg: "Config found!"
      when: config_file.stat.exists
      tags: lab9_when_registered
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab9_when_registered
```

### Lab 10 — `when` combined with a loop
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
      tags: lab10_when_loop
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab10_when_loop
```

---

## 3. Handlers — Run Tasks Only on Change, Only Once

### Lab 11 — Handler fires once even with two notifying tasks
```yaml
- name: test
  hosts: all
  tasks:
    - name: Copy nginx config
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx
      tags: lab11_handlers

    - name: Copy site config
      ansible.builtin.copy:
        src: site.conf
        dest: /etc/nginx/sites-available/site.conf
      notify: Restart nginx
      tags: lab11_handlers
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab11_handlers
```

### Lab 12 — Forcing handlers to run early (`flush_handlers`)
```yaml
- name: test
  hosts: all
  tasks:
    - name: Copy nginx config
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx
      tags: lab12_flush_handlers

    - name: Flush handlers now instead of at the end of the play
      ansible.builtin.meta: flush_handlers
      tags: lab12_flush_handlers

    - name: Continue with more tasks after handler already ran
      ansible.builtin.debug:
        msg: "Handler already fired above"
      tags: lab12_flush_handlers
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab12_flush_handlers
```

### Lab 13 — `listen`: one notify, many handlers
```yaml
- name: test
  hosts: all
  tasks:
    - name: Deploy new config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: "restart web stack"
      tags: lab13_listen
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
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab13_listen
```

### Lab 14 — Workshop exercise: Reload systemd handler
```yaml
- name: test
  hosts: all
  tasks:
    - name: Copy app systemd service file
      ansible.builtin.copy:
        src: myapp.service
        dest: /etc/systemd/system/myapp.service
      notify: Reload systemd
      tags: lab14_reload_systemd
  handlers:
    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: true
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab14_reload_systemd
```

---

## 4. Other Essentials

### Lab 15 — `register`: capture a task's result
```yaml
- name: test
  hosts: all
  tasks:
    - name: Check disk usage
      ansible.builtin.command: df -h /
      register: disk_usage
      changed_when: false
      tags: lab15_register

    - name: Show the result
      ansible.builtin.debug:
        var: disk_usage.stdout
      tags: lab15_register
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab15_register
```

### Lab 16 — `block` / `rescue` / `always`
```yaml
- name: test
  hosts: all
  tasks:
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
      tags: lab16_block_rescue_always
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab16_block_rescue_always
```

### Lab 17 — `tags`: run only part of a playbook
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
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags install
ansible-playbook -i inventory.ini labloop.yaml --skip-tags install
```

### Lab 18 — `vars` and `vars_files`
Requires a `secrets.yml` alongside `labloop.yaml`.
```yaml
- name: test
  hosts: all
  vars:
    app_port: 8080
  vars_files:
    - secrets.yml
  tasks:
    - name: Show the app port
      ansible.builtin.debug:
        msg: "App runs on port {{ app_port }}"
      tags: lab18_vars
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab18_vars
```

### Lab 19 — Facts
```yaml
- name: test
  hosts: all
  tasks:
    - name: Show the OS family
      ansible.builtin.debug:
        var: ansible_facts['os_family']
      tags: lab19_facts
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab19_facts
```

---

# Lab 2: Define Variables

## Method 1 — Variables in the playbook (`vars`)

### Lab 20
```yaml
- name: test
  hosts: all
  become: true
  vars:
    package_name: apache2
    service_name: apache2
    port: 80
  tasks:
    - name: Install package
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
      tags: lab20_vars_in_playbook

    - name: Start service
      ansible.builtin.systemd:
        name: "{{ service_name }}"
        state: started
      tags: lab20_vars_in_playbook

    - name: Display info
      ansible.builtin.debug:
        msg: "Service {{ service_name }} running on port {{ port }}"
      tags: lab20_vars_in_playbook
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab20_vars_in_playbook
```

## Method 2 — Variables in inventory (`host_vars`)

`inventory.ini` needs:
```ini
[local]
localhost ansible_connection=local ansible_user=amit app_name=myapp app_port=8080
```

### Lab 21
```yaml
- name: test
  hosts: all
  tasks:
    - name: Debug variables
      ansible.builtin.debug:
        msg: "App {{ app_name }} on port {{ app_port }}"
      tags: lab21_vars_inventory
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab21_vars_inventory
```

## Method 3 — External variable file (`group_vars`)

`group_vars/all.yml` (or `group_vars/<group>.yml`) needs:
```yaml
---
package_name: nginx
service_name: nginx
port: 80
admin_email: admin@example.com
```

### Lab 22
```yaml
- name: test
  hosts: all
  tasks:
    - name: Install and start service
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
      tags: lab22_group_vars
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab22_group_vars
```

## Method 4 — Command-line variables (`-e`)

### Lab 23
```yaml
- name: test
  hosts: all
  become: true
  vars:
    package_name: apache2
    service_name: apache2
    port: 80
  tasks:
    - name: Install package
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
      tags: lab23_cli_vars

    - name: Display info
      ansible.builtin.debug:
        msg: "Service {{ service_name }} running on port {{ port }}"
      tags: lab23_cli_vars
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab23_cli_vars -e "package_name=nginx port=8080"
```

## Method 5 — Variable file (`vars_files`)

`vars.yml` needs:
```yaml
---
package_name: apache2
service_name: apache2
port: 80
max_connections: 1000
enable_ssl: true
```

### Lab 24
```yaml
- name: test
  hosts: all
  become: true
  vars_files:
    - vars.yml
  tasks:
    - name: Install package
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
      tags: lab24_vars_files
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab24_vars_files
```

## Complete Lab Exercise

`vars.yml` needs:
```yaml
---
packages:
  - apache2
  - curl
  - wget
service_name: apache2
port: 80
state: present
```

### Lab 25
```yaml
- name: test
  hosts: all
  become: true
  vars_files:
    - vars.yml
  vars:
    environment_type: production
  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
      tags: lab25_complete_exercise

    - name: Install packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: "{{ state }}"
      loop: "{{ packages }}"
      tags: lab25_complete_exercise

    - name: Start service
      ansible.builtin.systemd:
        name: "{{ service_name }}"
        state: started
        enabled: yes
      tags: lab25_complete_exercise

    - name: Display variables
      ansible.builtin.debug:
        msg: |
          Environment: {{ environment_type }}
          Service: {{ service_name }}
          Port: {{ port }}
      tags: lab25_complete_exercise
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab25_complete_exercise
```

## Variable Types

### Lab 26 — String, list, and dictionary variables
```yaml
- name: test
  hosts: all
  vars:
    name: "apache2"
    description: "Web server"
    packages:
      - apache2
      - nginx
      - mysql-server
    server:
      name: webserver1
      ip: 192.168.1.10
      port: 80
  tasks:
    - name: Install packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
      tags: lab26_var_types

    - name: Display server info
      ansible.builtin.debug:
        msg: "Server {{ server.name }} at {{ server.ip }}:{{ server.port }}"
      tags: lab26_var_types
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab26_var_types
```

## Debugging Variables

### Lab 27 — All vars, a specific var, and a custom message
```yaml
- name: test
  hosts: all
  vars:
    package_name: apache2
  tasks:
    - name: Debug all variables
      ansible.builtin.debug:
        var: vars
      tags: lab27_debug_vars

    - name: Debug package name
      ansible.builtin.debug:
        var: package_name
      tags: lab27_debug_vars

    - name: Custom message
      ansible.builtin.debug:
        msg: "Installing {{ package_name }} on {{ ansible_hostname }}"
      tags: lab27_debug_vars
```
Run:
```bash
ansible-playbook -i inventory.ini labloop.yaml --tags lab27_debug_vars
```

---

## Variable Scope (Priority Order)

Lower to higher priority:

1. Role defaults
2. Group vars
3. Host vars
4. Task vars
5. Playbook vars
6. Command line (`-e`) — **highest priority, always wins**

## Quick Reference

| Lab | Topic | Tag |
|---|---|---|
| 1 | Basic loop | `lab1_basic_loop` |
| 2 | Loop over a variable | `lab2_loop_var` |
| 3 | Loop over dictionaries | `lab3_loop_dicts` |
| 4 | `loop_control` | `lab4_loop_control` |
| 5 | Loop exercise (directories) | `lab5_loop_dirs` |
| 6 | `when` basic | `lab6_when_basic` |
| 7 | `when` AND | `lab7_when_and` |
| 8 | `when` OR | `lab8_when_or` |
| 9 | `when` + registered var | `lab9_when_registered` |
| 10 | `when` + loop | `lab10_when_loop` |
| 11 | Handlers (notify once) | `lab11_handlers` |
| 12 | `flush_handlers` | `lab12_flush_handlers` |
| 13 | `listen` | `lab13_listen` |
| 14 | Reload systemd exercise | `lab14_reload_systemd` |
| 15 | `register` | `lab15_register` |
| 16 | `block/rescue/always` | `lab16_block_rescue_always` |
| 17 | `tags` | `install` / `config` |
| 18 | `vars` + `vars_files` | `lab18_vars` |
| 19 | Facts | `lab19_facts` |
| 20 | Vars: in playbook | `lab20_vars_in_playbook` |
| 21 | Vars: inventory | `lab21_vars_inventory` |
| 22 | Vars: group_vars | `lab22_group_vars` |
| 23 | Vars: CLI `-e` | `lab23_cli_vars` |
| 24 | Vars: vars_files | `lab24_vars_files` |
| 25 | Complete exercise | `lab25_complete_exercise` |
| 26 | Variable types | `lab26_var_types` |
| 27 | Debugging variables | `lab27_debug_vars` |
