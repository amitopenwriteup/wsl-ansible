# Ansible Lab Setup Guide

## Problem
Ansible playbook fails with permission error when trying to install packages using `apt` module.

**Error:**
```
E: Could not open lock file /var/lib/dpkg/lock-frontend - Permission denied
```

---

## Root Cause
- Conflicting become user settings in configuration files
- Passwordless sudo already configured but not being used properly
- Ansible trying to use wrong credentials

---

## Solution

### Step 1: Update inventory.ini
Remove password and become user settings.

**File: `inventory.ini`**
```ini
[local]
localhost ansible_connection=local ansible_user=amit
```

---

### Step 2: Update ansible.cfg
Simplify privilege escalation settings.

**File: `ansible.cfg`**
```ini
[defaults]
inventory = inventory.ini
host_key_checking = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

---

### Step 3: Keep Playbook as is
No changes needed to the playbook.

**File: `lab1.yaml`**
```yaml
- name: play1
  hosts: local
  become: true
  tasks:
    - name: Example from an Ansible Playbook
      ansible.builtin.ping:

    - name: Install apache httpd
      ansible.builtin.apt:
        name: apache2
        state: present
```

---

## Commands to Run

### Update files:
```bash
# Edit inventory.ini
nano inventory.ini

# Edit ansible.cfg
nano ansible.cfg
```

### Run playbook:
```bash
ansible-playbook -i inventory.ini lab1.yaml
```

### Check status:
```bash
# Verify apache installed
sudo systemctl status apache2

# Test apache
curl http://localhost
```

---


```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Still asking for password | Check `become_ask_pass = False` in ansible.cfg |
| Permission denied | Verify `amit ALL=(ALL) NOPASSWD: ALL` in sudoers |
| Host not found | Check `hosts: local` matches `[local]` in inventory |

---

## Quick Reference

```bash
# List all hosts
ansible all -i inventory.ini --list-hosts

# Run playbook with verbose output
ansible-playbook -i inventory.ini lab1.yaml -v

# Dry run (check mode)
ansible-playbook -i inventory.ini lab1.yaml --check
```
