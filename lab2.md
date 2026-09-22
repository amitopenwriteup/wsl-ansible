# Ansible Lab 2: Define Variables

## Overview
Learn different ways to define and use variables in Ansible playbooks.

---

## Variable Definition Methods

### Method 1: In Playbook (vars)

**File: `lab2.yaml`**
```yaml
- name: play1
  hosts: local
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
    
    - name: Start service
      ansible.builtin.systemd:
        name: "{{ service_name }}"
        state: started
    
    - name: Display info
      ansible.builtin.debug:
        msg: "Service {{ service_name }} running on port {{ port }}"
```

---

### Method 2: In Inventory (host_vars)

**File: `inventory.ini`**
```ini
[local]
localhost ansible_connection=local ansible_user=amit app_name=myapp app_port=8080
```

**Usage in playbook:**
```yaml
- name: play1
  hosts: local
  tasks:
    - name: Debug variables
      ansible.builtin.debug:
        msg: "App {{ app_name }} on port {{ app_port }}"
```

---

### Method 3: External Variable File (group_vars)

**Directory structure:**
```
.
├── inventory.ini
├── lab2.yaml
└── group_vars/
    └── local.yml
```

**File: `group_vars/local.yml`**
```yaml
---
package_name: nginx
service_name: nginx
port: 80
admin_email: admin@example.com
```

**Usage in playbook:**
```yaml
- name: play1
  hosts: local
  tasks:
    - name: Install and start service
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
```

---

### Method 4: Command Line Variables

**Run playbook with variables:**
```bash
ansible-playbook lab2.yaml -e "package_name=nginx port=8080"
```

**Multiple variables:**
```bash
ansible-playbook lab2.yaml -e "package_name=nginx port=8080 service_name=nginx"
```

---

### Method 5: Variable File (vars_files)

**File: `vars.yml`**
```yaml
---
package_name: apache2
service_name: apache2
port: 80
max_connections: 1000
enable_ssl: true
```

**File: `lab2.yaml`**
```yaml
- name: play1
  hosts: local
  become: true
  vars_files:
    - vars.yml
  
  tasks:
    - name: Install package
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
```

---

## Complete Lab Exercise

### Step 1: Create Variables File

**File: `vars.yml`**
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

---

### Step 2: Create Playbook

**File: `lab2.yaml`**
```yaml
- name: play1
  hosts: local
  become: true
  vars_files:
    - vars.yml
  
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
          Port: {{ port }}
```

---

### Step 3: Update Inventory

**File: `inventory.ini`**
```ini
[local]
localhost ansible_connection=local ansible_user=amit
```

---

### Step 4: Run Playbook

```bash
# Basic run
ansible-playbook -i inventory.ini lab2.yaml

# With extra variables
ansible-playbook -i inventory.ini lab2.yaml -e "port=8080"

# Verbose output
ansible-playbook -i inventory.ini lab2.yaml -v
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
    port: 80
```

### Using Lists
```yaml
tasks:
  - name: Install packages
    ansible.builtin.apt:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"
```

### Using Dictionaries
```yaml
tasks:
  - name: Display server info
    ansible.builtin.debug:
      msg: "Server {{ server.name }} at {{ server.ip }}:{{ server.port }}"
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
ansible-playbook lab2.yaml -e "package_name=nginx"  # This wins!
```

---

## Debugging Variables

### Print all variables
```yaml
- name: Debug all variables
  ansible.builtin.debug:
    var: vars
```

### Print specific variable
```yaml
- name: Debug package name
  ansible.builtin.debug:
    var: package_name
```

### Print with message
```yaml
- name: Custom message
  ansible.builtin.debug:
    msg: "Installing {{ package_name }} on {{ ansible_hostname }}"
```

---

## Quick Reference

| Method | Location | Priority |
|--------|----------|----------|
| vars | Playbook | Medium |
| inventory | inventory.ini | Low |
| group_vars | group_vars/ | Medium-Low |
| host_vars | host_vars/ | Medium-Low |
| vars_files | External file | Medium |
| -e flag | Command line | Highest |

---

## Commands to Run

```bash
# Create directory for group vars
mkdir -p group_vars

# Create variables file
nano vars.yml

# Create playbook
nano lab2.yaml

# Run playbook
ansible-playbook -i inventory.ini lab2.yaml

# Run with extra variables
ansible-playbook -i inventory.ini lab2.yaml -e "port=8080"

# List all variables
ansible-playbook -i inventory.ini lab2.yaml -e "port=8080" --extra-vars="debug=true"
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

1. **Create** `group_vars/local.yml` with variables
2. **Create** `lab2.yaml` playbook
3. **Run** playbook with default variables
4. **Run** playbook with `-e` flag to override variables
5. **Add** debug tasks to display all variables
6. **Verify** priority (command line overrides file variables)
