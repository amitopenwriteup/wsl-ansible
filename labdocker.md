# Workshop: Install Docker with Ansible (RedHat and Debian families)

## Objective

Write and run an Ansible playbook that installs Docker CE on any managed host, choosing the right method based on `ansible_os_family` (RedHat/CentOS/Rocky/Alma or Debian/Ubuntu).

**Duration:** ~45 minutes
**Level:** Beginner to intermediate

---

## Prerequisites

| Item | Requirement |
|------|-------------|
| Control node | Linux/macOS with Ansible installed (`ansible --version`) |
| Managed nodes | 1+ RedHat-family or Ubuntu hosts, reachable over SSH |
| Access | SSH key-based login and `sudo` rights on managed nodes |
| Editor | `vi` / `vim` |

Install Ansible on the control node if needed:

```bash
# RedHat family
sudo dnf install -y ansible-core

# Debian / Ubuntu
sudo apt update && sudo apt install -y ansible
```

---

## Lab 1: Create the project directory

```bash
mkdir -p ~/ansible-docker-lab
cd ~/ansible-docker-lab
```

> No `-i inventory.ini` is used in this workshop. Ansible will use the default inventory (`/etc/ansible/hosts`, or whatever `ANSIBLE_INVENTORY` / `ansible.cfg` points to). Point that at your RedHat and Ubuntu hosts before running the playbook, or pass an inventory path explicitly with `-i <path>` if you keep one elsewhere.

---

### vi quick reference

| Key / Command | Action |
|---------------|--------|
| `i` | Enter **insert mode** (start typing) |
| `Esc` | Return to **normal mode** |
| `:set paste` | Turn on paste mode (prevents auto-indent breaking YAML) |
| `:set nopaste` | Turn paste mode off |
| `:set number` | Show line numbers |
| `:w` | Save |
| `:wq` or `ZZ` | Save and quit |
| `:q!` | Quit without saving |
| `dd` | Delete current line |
| `yy` / `p` | Copy line / paste |
| `u` | Undo |
| `/text` then `n` | Search for text, jump to next match |
| `gg` / `G` | Jump to first / last line |
| `:%s/old/new/g` | Replace all occurrences |

---

## Lab 2: Create the playbook (using vi)

Open the file:

```bash
vi install_docker.yml
```

1. Type `:set paste` and press `Enter` (avoids indentation problems when pasting).
2. Press `i` to enter insert mode.
3. Paste the playbook below.
4. Press `Esc`, then type `:set nopaste`.
5. Save and exit with `:wq`.

```yaml
---
- name: Install Docker based on OS family
  hosts: all
  become: true
  vars:
    docker_packages_redhat:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    docker_packages_debian:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    docker_repo_url_redhat: https://download.docker.com/linux/centos/docker-ce.repo
    docker_repo_url_debian: https://download.docker.com/linux/ubuntu
    docker_repo_key_url_debian: https://download.docker.com/linux/ubuntu/gpg
  tasks:
    - name: Install prerequisites on RedHat
      dnf:
        name: dnf-plugins-core
        state: present
      when: ansible_os_family == "RedHat"
    - name: Add Docker CE repo on RedHat
      get_url:
        url: "{{ docker_repo_url_redhat }}"
        dest: /etc/yum.repos.d/docker-ce.repo
      when: ansible_os_family == "RedHat"
      notify: Restart Docker
    - name: Install Docker packages on RedHat
      dnf:
        name: "{{ docker_packages_redhat }}"
        state: present
      when: ansible_os_family == "RedHat"
      notify: Restart Docker
    - name: Install prerequisites on Debian
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
      when: ansible_os_family == "Debian"
    - name: Add Docker GPG key on Debian
      apt_key:
        url: "{{ docker_repo_key_url_debian }}"
        state: present
      when: ansible_os_family == "Debian"
    - name: Add Docker repository on Debian
      apt_repository:
        repo: "deb [arch=amd64] {{ docker_repo_url_debian }} {{ ansible_lsb.codename }} stable"
        state: present
      when: ansible_os_family == "Debian"
      notify: Restart Docker

    - name: Install Docker packages on Debian
      apt:
        name: "{{ docker_packages_debian }}"
        state: present
      when: ansible_os_family == "Debian"
      notify: Restart Docker

    - name: Enable and start Docker service
      systemd:
        name: docker
        enabled: true
        state: started

  handlers:
    - name: Restart Docker
      systemd:
        name: docker
        state: restarted
```

### Verify the file inside vi

- `:set number` shows line numbers (helpful when Ansible reports "line X").
- `/apt_key` jumps to that task.
- YAML uses **spaces only**, never tabs. Check with `:set list` (tabs show as `^I`).

---

## Lab 3: Understand the playbook

| Section | What it does |
|---------|--------------|
| `hosts: all` | Runs against every host in the inventory |
| `become: true` | Uses sudo/root for all tasks |
| `vars` | Package lists and repo URLs per OS family |
| `when: ansible_os_family == ...` | Runs a task only on matching OS |
| `notify` | Queues the handler when a task changes something |
| `handlers` | Run **once at the end** of the play, only if notified |
| `systemd` task | Ensures Docker starts now and on boot |

**Flow on RedHat:** install `dnf-plugins-core`, download the repo file, install Docker packages, start the service.

**Flow on Debian/Ubuntu:** install prerequisites, add the GPG key, add the apt repo, install Docker packages, start the service.

---

## Lab 4: Validate and run

### 1. Syntax check

```bash
ansible-playbook install_docker.yml --syntax-check
```

### 2. Dry run (no changes made)

```bash
ansible-playbook install_docker.yml --check
```

> Note: `--check` may report errors on later tasks because earlier packages are not really installed yet. That is normal.

### 3. Run the playbook

```bash
ansible-playbook install_docker.yml
```

Useful variations:

```bash
# Verbose output
ansible-playbook install_docker.yml -v

# Only one host
ansible-playbook install_docker.yml --limit ubuntu01

# Ask for sudo password
ansible-playbook install_docker.yml --ask-become-pass
```

### 4. Run it again (idempotency test)

A second run should show `changed=0`. That confirms the playbook is idempotent.

---

## Lab 5: Verify the installation

Ad-hoc commands from the control node:

```bash
ansible all -b -m command -a "docker --version"
ansible all -b -m command -a "docker compose version"
ansible all -b -m command -a "systemctl is-active docker"
ansible all -b -m command -a "docker run --rm hello-world"
```

Or log in to a managed host and check directly:

```bash
ssh ubuntu@192.168.1.11
sudo docker --version
sudo systemctl status docker
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `mapping values are not allowed here` | YAML indentation or tabs | Open in vi, run `:set list`, replace tabs with spaces |
| `ansible_lsb` is undefined on Debian | `lsb-release` not installed before facts were gathered | Use `{{ ansible_distribution_release }}` instead of `{{ ansible_lsb.codename }}` |
| `apt_key` deprecation warning | `apt-key` is deprecated on newer Ubuntu | Store the key in `/etc/apt/keyrings` and reference it with `signed-by=` in the repo line instead of using `apt_key` |
| `Permission denied (publickey)` | Wrong user or key | Check `ansible_user` and `ansible_ssh_private_key_file` |
| `sudo: a password is required` | Passwordless sudo not set | Add `--ask-become-pass` |
| Repo not found on Rocky/Alma | Repo path differs | Use the `centos` repo URL (as in this playbook) or the `rhel` one for RHEL |

---

## Cleanup (optional)

```bash
ansible all -b -m package -a "name=docker-ce,docker-ce-cli,containerd.io state=absent"
```

---

## Summary

- Created an inventory and playbook using `vi`.
- Used `ansible_os_family` conditionals to support RedHat and Debian hosts.
- Used `notify` and handlers to restart Docker only when something changed.
- Validated with `--syntax-check`, `--check`, and an idempotency re-run.
