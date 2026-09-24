# Deploy PostgreSQL with Ansible using `community.docker`

## Overview

This playbook uses the `community.docker` collection to create a Docker volume, pull the official PostgreSQL image, and run a PostgreSQL container — instead of shelling out to raw `docker` commands.

---

## Prerequisites

### 1. Install the collection on the control node

```bash
ansible-galaxy collection install community.docker
```

Verify it installed:

```bash
ansible-galaxy collection list | grep community.docker
```

### 2. Install the Python Docker SDK on the managed node

The `community.docker` modules talk to the Docker daemon through the Docker SDK for Python, not the `docker` CLI, so it must be present on the **managed node** (the host running the container), not the control node.

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install -y python3-pip
sudo pip3 install docker

# RedHat / CentOS / Rocky
sudo dnf install -y python3-pip
sudo pip3 install docker
```

Or install it via Ansible against the managed node:

```bash
ansible postgres_host -b -m pip -a "name=docker state=present"
```

### 3. Confirm Docker itself is installed and running on the managed node

```bash
ansible postgres_host -b -m command -a "docker --version"
ansible postgres_host -b -m command -a "systemctl is-active docker"
```

If Docker isn't installed yet, install it first (see your Docker-install playbook/role) before running this one.

> Note: `postgres_host` must resolve to a host or group in whatever inventory your `ansible-playbook` command picks up (default inventory, `ansible.cfg`, or `ANSIBLE_INVENTORY`).

---

## The playbook (using vi)

```bash
vi postgres.yml
```

Type `:set paste` and press `Enter` first — this prevents `vi`'s autoindent from mangling the YAML or appending stray characters to a line (this is what caused the `gpg---` URL corruption earlier). Then press `i` to enter insert mode, paste the content below, press `Esc`, type `:set nopaste`, then `:wq` to save.

```yaml
---
- name: Deploy PostgreSQL using Docker collection
  hosts: postgres_host
  become: true
  tasks:
    - name: Create a Docker volume for PostgreSQL data
      community.docker.docker_volume:
        name: pgdata

    - name: Pull the official PostgreSQL image
      community.docker.docker_image:
        name: postgres
        source: pull

    - name: Run PostgreSQL container
      community.docker.docker_container:
        name: postgres
        image: postgres
        state: started
        restart_policy: always
        ports:
          - "5432:5432"
        env:
          POSTGRES_PASSWORD: mysecretpassword
          POSTGRES_USER: myuser
          POSTGRES_DB: mydb
        volumes:
          - pgdata:/var/lib/postgresql/data
```

---

## What each task does

| Task | Module | Purpose |
|------|--------|---------|
| Create a Docker volume | `docker_volume` | Creates the named volume `pgdata` so PostgreSQL data survives container restarts/removal |
| Pull the official PostgreSQL image | `docker_image` | Pulls `postgres:latest` from Docker Hub |
| Run PostgreSQL container | `docker_container` | Starts a container named `postgres`, maps port `5432`, sets DB credentials via env vars, mounts `pgdata` at the Postgres data directory, and restarts the container automatically (`restart_policy: always`) unless you stop it manually |

---

## Run it

```bash
ansible-playbook postgres.yml --syntax-check
ansible-playbook postgres.yml --check --diff
ansible-playbook postgres.yml
```

---

## Verify

```bash
ansible postgres_host -b -m command -a "docker ps"
ansible postgres_host -b -m command -a "docker volume ls"
ansible postgres_host -b -m command -a "docker exec postgres pg_isready -U myuser"
```

Or connect directly from the managed node:

```bash
ssh ubuntu@192.168.1.20
docker exec -it postgres psql -U myuser -d mydb
```

---

## Notes and hardening ideas

- **Password in plain text:** `POSTGRES_PASSWORD` is hardcoded here. For anything beyond a lab, move it into an Ansible Vault-encrypted variable instead of committing it in plaintext:
  ```bash
  ansible-vault create group_vars/postgres_host/vault.yml
  ```
  then reference it as `env: { POSTGRES_PASSWORD: "{{ vault_postgres_password }}" }`.
- **Idempotency:** re-running the playbook won't recreate the volume or re-pull the image unless something changed, and `docker_container` won't restart the container unless its config actually differs.
- **Port exposure:** `5432:5432` publishes Postgres to every interface on the host. Restrict with a firewall rule or bind to localhost only (`127.0.0.1:5432:5432`) if the DB shouldn't be reachable externally.
- **Image tag:** `image: postgres` pulls `latest`, which can change under you. Pin a version, e.g. `postgres:16`, for reproducible deployments.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `Failed to import docker or docker-py` | Docker SDK not installed on the managed node — install with `pip3 install docker` there |
| `couldn't connect to Docker daemon` | Docker service not running on managed node, or user lacks permission — check `systemctl status docker` |
| `docker_container` reports changed every run | A field like `env` or `ports` differs slightly from what's already running — check `docker inspect postgres` against your playbook values |
| Connection refused on port 5432 | Firewall blocking the port, or container failed to start — check `docker logs postgres` |
| `postgres_host` has no hosts | Inventory group name doesn't match — check `ansible-inventory --list` |
