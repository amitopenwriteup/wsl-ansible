# Ansible Lab 2: Define Variables

## Overview
Learn different ways to define and use variables in Ansible playbooks.

> **Note:** `port` is a reserved variable name in Ansible (it collides with the
> connection variable `ansible_port`), which triggers:
> `[WARNING]: Found variable using reserved name: port`
> Every example below uses `http_port` instead to avoid that warning.

---

## Variable Definition Methods

### Method 1: In Playbook (vars)

```bash
vi lab2_method1.yaml
```

```yaml
- name: play1
  hosts: local
  become: true
  vars:
    package_name: apache2
    service_name: apache2
    http_port: 80

  tasks:
    - name: Install package
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
    - name: Display info
      ansible.builtin.debug:
        msg: "Service {{ service_name }} running on port {{ http_port }}"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab2_method1.yaml
```

---

### Method 2: In Inventory (host_vars)

```bash
vi inventory.ini
```

```ini
[local]
localhost ansible_connection=local ansible_user=amit app_name=myapp app_port=8080
```

**Usage in playbook:**
```bash
vi lab2_method2.yaml
```

```yaml
- name: play1
  hosts: local
  tasks:
    - name: Debug variables
      ansible.builtin.debug:
        msg: "App {{ app_name }} on port {{ app_port }}"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab2_method2.yaml
```

---

### Method 3: External Variable File (group_vars)

**Directory structure:**
```
.
├── inventory.ini
├── lab2_method3.yaml
└── group_vars/
    └── local.yml
```

```bash
mkdir group_vars
vi group_vars/local.yml
```

```yaml
---
package_name: nginx
service_name: nginx
http_port: 80
admin_email: admin@example.com
```

**Usage in playbook:**
```bash
vi lab2_method3.yaml
```

```yaml
- name: play1
  hosts: local
  tasks:
    - name: Install and start service
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab2_method3.yaml
```

---

### Method 4: Command Line Variables

```bash
vi lab2_method4.yaml
```

```yaml
- name: play1
  hosts: local
  become: true
  tasks:
    - name: Install package
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
    - name: Display info
      ansible.builtin.debug:
        msg: "Service {{ service_name }} running on port {{ http_port }}"
```

**Run playbook with variables:**
```bash
ansible-playbook -i inventory.ini lab2_method4.yaml -e "package_name=nginx http_port=8080"
```

**Multiple variables:**
```bash
ansible-playbook -i inventory.ini lab2_method4.yaml -e "package_name=nginx http_port=8080 service_name=nginx"
```

---

### Method 5: Variable File (vars_files)

```bash
vi vars_method5.yml
```

```yaml
---
package_name: apache2
service_name: apache2
http_port: 80
max_connections: 1000
enable_ssl: true
```

```bash
vi lab2_method5.yaml
```

```yaml
- name: play1
  hosts: local
  become: true
  vars_files:
    - vars_method5.yml

  tasks:
    - name: Install package
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab2_method5.yaml
```

---

## Complete Lab Exercise

### Step 1: Create Variables File

```bash
vi vars_complete.yml
```

```yaml
---
packages:
  - apache2
  - curl
  - wget

service_name: apache2
http_port: 80
state: present
```

---

### Step 2: Create Playbook

```bash
vi lab2_complete.yaml
```

```yaml
- name: play1
  hosts: local
  become: true
  vars_files:
    - vars_complete.yml

  vars:
    environment_type: production

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes

    - name: Install packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: "{{ state }}"
      loop: "{{ packages }}"

    - name: Start service
      ansible.builtin.systemd:
        name: "{{ service_name }}"
        state: started
        enabled: yes

    - name: Display variables
      ansible.builtin.debug:
        msg: |
          Environment: {{ environment_type }}
          Service: {{ service_name }}
          Port: {{ http_port }}
```

---

### Step 3: Update Inventory

```bash
vi inventory.ini
```

```ini
[local]
localhost ansible_connection=local ansible_user=amit
```

---

### Step 4: Run Playbook

```bash
# Basic run
ansible-playbook -i inventory.ini lab2_complete.yaml

# With extra variables
ansible-playbook -i inventory.ini lab2_complete.yaml -e "http_port=8080"

# Verbose output
ansible-playbook -i inventory.ini lab2_complete.yaml -v
```

---

## Variable Types

### String Variables
```yaml
vars:
  name: "apache2"
  description: "Web server"
```

### List Variables
```yaml
vars:
  packages:
    - apache2
    - nginx
    - mysql-server
```

### Dictionary Variables
```yaml
vars:
  server:
    name: webserver1
    ip: 192.168.1.10
    http_port: 80
```

### Using Lists and Dictionaries Together

```bash
vi lab2_vartypes.yaml
```

```yaml
- name: play1
  hosts: local
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
      http_port: 80

  tasks:
    - name: Install packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"

    - name: Display server info
      ansible.builtin.debug:
        msg: "Server {{ server.name }} at {{ server.ip }}:{{ server.http_port }}"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab2_vartypes.yaml
```

---

## Variable Scope (Priority Order)

**Lower to Higher Priority:**
1. Role defaults
2. Group vars
3. Host vars
4. Task vars
5. Playbook vars
6. Command line (-e)

**Example:**
```bash
# Command line has HIGHEST priority
ansible-playbook -i inventory.ini lab2_method1.yaml -e "package_name=nginx"  # This wins!
```

---

## Debugging Variables

```bash
vi lab2_debug.yaml
```

```yaml
- name: play1
  hosts: local
  vars:
    package_name: apache2
  tasks:
    - name: Debug all variables
      ansible.builtin.debug:
        var: vars

    - name: Debug package name
      ansible.builtin.debug:
        var: package_name

    - name: Custom message
      ansible.builtin.debug:
        msg: "Installing {{ package_name }} on {{ ansible_hostname }}"
```

**Run:**
```bash
ansible-playbook -i inventory.ini lab2_debug.yaml
```

---

## Quick Reference

| Method | File | Location | Priority |
|--------|------|----------|----------|
| vars | lab2_method1.yaml | Playbook | Medium |
| inventory | lab2_method2.yaml | inventory.ini | Low |
| group_vars | lab2_method3.yaml | group_vars/ | Medium-Low |
| host_vars | — | host_vars/ | Medium-Low |
| -e flag | lab2_method4.yaml | Command line | Highest |
| vars_files | lab2_method5.yaml | External file | Medium |
| complete exercise | lab2_complete.yaml | vars_files + vars | — |

---

## Commands to Run

```bash
# Create directory for group vars
mkdir -p group_vars

# Create variables files
vi vars_method5.yml
vi vars_complete.yml
vi group_vars/local.yml

# Create playbooks
vi lab2_method1.yaml
vi lab2_method2.yaml
vi lab2_method3.yaml
vi lab2_method4.yaml
vi lab2_method5.yaml
vi lab2_complete.yaml
vi lab2_vartypes.yaml
vi lab2_debug.yaml

# Run a playbook
ansible-playbook -i inventory.ini lab2_complete.yaml

# Run with extra variables
ansible-playbook -i inventory.ini lab2_complete.yaml -e "http_port=8080"

# List all variables
ansible-playbook -i inventory.ini lab2_complete.yaml -e "http_port=8080" --extra-vars="debug=true"
```

---

## Expected Output

```
TASK [Install packages] ***
ok: [localhost] => (item=apache2)
ok: [localhost] => (item=curl)
ok: [localhost] => (item=wget)

TASK [Start service] ***
changed: [localhost]

TASK [Display variables] ***
ok: [localhost] => {
    "msg": "Environment: production\nService: apache2\nPort: 80"
}

PLAY RECAP ***
localhost : ok=5  changed=1  unreachable=0  failed=0
```

---

## Practice Exercise

1. **Create** `group_vars/local.yml` with variables (`vi group_vars/local.yml`)
2. **Create** `lab2_method3.yaml` playbook (`vi lab2_method3.yaml`)
3. **Run** playbook with default variables: `ansible-playbook -i inventory.ini lab2_method3.yaml`
4. **Run** playbook with `-e` flag to override variables: `ansible-playbook -i inventory.ini lab2_method3.yaml -e "http_port=8080"`
5. **Add** debug tasks to display all variables (see `lab2_debug.yaml`)
6. **Verify** priority (command line overrides file variables)
