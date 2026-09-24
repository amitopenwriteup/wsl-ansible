# Workshop: Build an Ansible Role for Docker Install

## Objective

Turn the Docker install logic into a reusable **role** with a clean structure:

```
roles/docker/
├── tasks/
│   ├── main.yml      # decides which OS file to run
│   ├── debian.yml    # Debian/Ubuntu install steps
│   └── redhat.yml    # RedHat/CentOS/Rocky install steps
├── handlers/
│   └── main.yml      # restart docker
└── vars/
    └── main.yml       # package lists & repo URLs
```

**Duration:** ~40 minutes
**Level:** Intermediate
**Prerequisite:** Ansible installed, at least one Debian-family or RedHat-family host reachable over SSH with sudo.

---

## Lab 1: Scaffold the role

```bash
mkdir -p ~/ansible-docker-role && cd ~/ansible-docker-role
ansible-galaxy init roles/docker
```

This creates the standard role skeleton (`tasks/`, `handlers/`, `vars/`, `defaults/`, `meta/`, etc.). We'll only fill in what we need and can leave the rest empty.

Check what was created:

```bash
find roles/docker -type f
```

---

## Lab 2: Define variables (`vars/main.yml`)

```bash
vi roles/docker/vars/main.yml
```

Press `i`, paste, then `Esc` `:wq`:

```yaml
---
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
```

---

## Lab 3: RedHat tasks (`tasks/redhat.yml`)

```bash
vi roles/docker/tasks/redhat.yml
```

```yaml
---
- name: Install prerequisites on RedHat
  dnf:
    name: dnf-plugins-core
    state: present

- name: Add Docker CE repo on RedHat
  get_url:
    url: "{{ docker_repo_url_redhat }}"
    dest: /etc/yum.repos.d/docker-ce.repo
  notify: Restart Docker

- name: Install Docker packages on RedHat
  dnf:
    name: "{{ docker_packages_redhat }}"
    state: present
  notify: Restart Docker
```

Notice there's no `when:` here — the OS check happens once in `tasks/main.yml`, so these files stay clean.

---

## Lab 4: Debian tasks (`tasks/debian.yml`)

```bash
vi roles/docker/tasks/debian.yml
```

```yaml
---
- name: Install prerequisites on Debian
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
    state: present

- name: Add Docker GPG key on Debian
  apt_key:
    url: "{{ docker_repo_key_url_debian }}"
    state: present

- name: Add Docker repository on Debian
  apt_repository:
    repo: "deb [arch=amd64] {{ docker_repo_url_debian }} {{ ansible_distribution_release }} stable"
    state: present
  notify: Restart Docker

- name: Install Docker packages on Debian
  apt:
    name: "{{ docker_packages_debian }}"
    state: present
    update_cache: true
  notify: Restart Docker
```

---

## Lab 5: Main task file (`tasks/main.yml`)

This is the only place that checks `ansible_os_family`, and it includes the right file:

```bash
vi roles/docker/tasks/main.yml
```

```yaml
---
- name: Run RedHat-specific tasks
  include_tasks: redhat.yml
  when: ansible_os_family == "RedHat"

- name: Run Debian-specific tasks
  include_tasks: debian.yml
  when: ansible_os_family == "Debian"

- name: Enable and start Docker service
  systemd:
    name: docker
    enabled: true
    state: started
```

---

## Lab 6: Handler (`handlers/main.yml`)

```bash
vi roles/docker/handlers/main.yml
```

```yaml
---
- name: Restart Docker
  systemd:
    name: docker
    state: restarted
```

---

## Lab 7: Write the playbook that uses the role

```bash
vi site.yml
```

```yaml
---
- name: Install Docker via role
  hosts: all
  become: true
  roles:
    - docker
```

Your final layout:

```
ansible-docker-role/
├── site.yml
└── roles/
    └── docker/
        ├── tasks/
        │   ├── main.yml
        │   ├── debian.yml
        │   └── redhat.yml
        ├── handlers/
        │   └── main.yml
        └── vars/
            └── main.yml
```

---

## Lab 8: Test it

### 1. Syntax check

```bash
ansible-playbook site.yml --syntax-check
```

### 2. List the tasks the role will run (no connection needed)

```bash
ansible-playbook site.yml --list-tasks
```

### 3. Dry run against real hosts

```bash
ansible-playbook site.yml --check --diff
```

### 4. Run it for real

```bash
ansible-playbook site.yml
```

### 5. Idempotency check — run it again

```bash
ansible-playbook site.yml
```

Expect `changed=0` for every host the second time.

### 6. Verify Docker is installed

```bash
ansible all -b -m command -a "docker --version"
ansible all -b -m command -a "systemctl is-active docker"
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `role 'docker' not found` | Run the playbook from `~/ansible-docker-role` so `roles/` is next to `site.yml`, or set `roles_path` in `ansible.cfg` |
| `include_tasks` file not found | Check the filename matches exactly (`debian.yml` / `redhat.yml`) and sits under `roles/docker/tasks/` |
| Vars not found (`docker_packages_debian` undefined) | Confirm the content is under `roles/docker/vars/main.yml` — vars files must be named `main.yml` to auto-load |
| Handler never fires | Handler name in `notify:` must match the handler's `name:` exactly, including case |
| YAML errors | Open the file in `vi`, run `:set list` to check for stray tabs, `:set number` to match error line numbers |

---

## Summary

- Split OS-specific logic into `tasks/debian.yml` and `tasks/redhat.yml`, keeping `tasks/main.yml` as a simple dispatcher.
- Centralized package lists and URLs in `vars/main.yml`.
- Used a single handler in `handlers/main.yml`, triggered by `notify` from either OS path.
- Validated with `--syntax-check`, `--list-tasks`, `--check --diff`, and a real + repeat run to confirm idempotency.
