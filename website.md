# Lab: Deploy and Validate a Website Using Dockerized NGINX on Ubuntu via Ansible

**Topic:** Ansible – Docker containers, file/copy modules, validation
**Target OS:** Ubuntu 22.04 / 24.04 LTS
**Estimated time:** 45–60 minutes

---

## 1. Objective

In this lab you will use Ansible to:

- Create a website directory on a remote Ubuntu host
- Copy a static `index.html` to it
- Run an **NGINX container** with that directory mounted as its web root
- Wait for the service to come up and **validate** that the site responds

---

## 2. Lab Architecture

```text
Control node (Ansible)                 Remote host (Ubuntu)
┌──────────────────────┐   SSH        ┌──────────────────────────────┐
│ inventory.ini        │ ───────────► │ /opt/website/index.html      │
│ files/index.html     │              │        │ (bind mount, ro)    │
│ deploy_website.yaml  │              │        ▼                     │
└──────────────────────┘              │ Docker: nginx-site           │
                                      │ host:81 ──► container:80     │
                                      └──────────────────────────────┘
```

---

## 3. Prerequisites

| Requirement | Where |
|---|---|
| Ansible 2.14+ | Control node |
| SSH access with a sudo-capable user (e.g. `ubuntu`) | Control node → remote host |
| Ubuntu host with internet access | Remote host |
| Docker Engine installed and running | Remote host |
| Python Docker SDK (`python3-docker`) | Remote host |
| `community.docker` collection | Control node |

### 3.1 Install Docker on the Ubuntu host (if not already installed)

Run on the remote host:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo docker --version
```

### 3.2 Install the Python Docker SDK on the remote host

The `community.docker.docker_container` module needs it:

```bash
sudo apt install -y python3-docker
```

---

## 4. Step-by-Step Setup

### Step 1 – Create the project directory

```bash
mkdir ansible-docker-website && cd ansible-docker-website
```

### Step 2 – Create the inventory

`inventory.ini`:

```ini
[remotes]
ubuntu01 ansible_host=192.168.56.20 ansible_user=ubuntu
```

Test connectivity:

```bash
ansible -i inventory.ini remotes -m ping
```

### Step 3 – Create `index.html` content

```bash
mkdir files
```

`files/index.html`:

```html
<!DOCTYPE html>
<html>
  <body>
    <h1>Hello from Ansible</h1>
  </body>
</html>
```

### Step 4 – Main playbook: `deploy_website.yaml`

```yaml
- name: Deploy and validate a website using Docker NGINX
  hosts: remotes
  become: yes
  vars:
    website_path: /opt/website
    site_url: http://localhost:81
    test_file_path: /tmp/test-index.html
    expected_text: "Hello from Ansible"
  tasks:
    - name: Create website directory
      file:
        path: "{{ website_path }}"
        state: directory
        mode: '0755'

    - name: Copy website files to remote host
      copy:
        src: files/index.html
        dest: "{{ website_path }}/index.html"

    - name: Run NGINX container with mounted website directory
      community.docker.docker_container:
        name: nginx-site
        image: nginx
        state: started
        restart_policy: always
        ports:
          - "81:80"
        volumes:
          - "{{ website_path }}:/usr/share/nginx/html:ro"

    - name: Wait for NGINX to be available on port 81
      wait_for:
        port: 81
        timeout: 30

    - name: Fetch the index page
      get_url:
        url: "{{ site_url }}"
        dest: "{{ test_file_path }}"
        force: yes

    - name: Display success message
      debug:
        msg: "Website deployed and validated successfully!"
```

### Step 5 – Install required collections

```bash
ansible-galaxy collection install community.docker
```

### Step 6 – Run the playbook

```bash
ansible-playbook -i inventory.ini deploy_website.yaml
```

Expected result: all tasks finish with `ok` or `changed`, and the final task prints the success message.

---

## 5. Task Breakdown (What Each Task Does)

| # | Task | Module | Purpose |
|---|---|---|---|
| 1 | Create website directory | `file` | Creates `/opt/website` with mode `0755` |
| 2 | Copy website files | `copy` | Places `index.html` in the web root |
| 3 | Run NGINX container | `community.docker.docker_container` | Starts `nginx`, maps host port 81 to 80, mounts the site read-only |
| 4 | Wait for port 81 | `wait_for` | Pauses until NGINX is listening (max 30 s) |
| 5 | Fetch the index page | `get_url` | Downloads the page to `/tmp/test-index.html` to prove it is served |
| 6 | Display success message | `debug` | Prints a confirmation |

---

## 6. Verify the Deployment

On the remote host:

```bash
sudo docker ps --filter name=nginx-site
curl http://localhost:81
cat /tmp/test-index.html
```

From the control node:

```bash
ansible -i inventory.ini remotes -b -m command -a "docker ps --filter name=nginx-site"
ansible -i inventory.ini remotes -m uri -a "url=http://localhost:81 return_content=yes"
```

You should see `Hello from Ansible` in the output.

If UFW is enabled on the Ubuntu host and you want to reach the site from another machine, allow the port:

```bash
sudo ufw status
sudo ufw allow 81/tcp
```

---

## 7. Exercises

### Exercise 1 – Idempotency

Run the playbook a second time. Which tasks report `changed` and which report `ok`? Explain why.

*Hint: `get_url` with `force: yes` re-downloads the file on every run.*

### Exercise 2 – Make the validation real

The playbook defines `expected_text` but never uses it, so a wrong page would still "pass". Replace the last two tasks with a check that fails if the text is missing:

```yaml
    - name: Validate page content
      ansible.builtin.uri:
        url: "{{ site_url }}"
        return_content: true
        status_code: 200
      register: site_response
      failed_when: expected_text not in site_response.content

    - name: Display success message
      ansible.builtin.debug:
        msg: "Website deployed and validated successfully!"
```

Then change `expected_text` to something wrong and confirm the playbook fails.

### Exercise 3 – Update the site

Edit `files/index.html`, rerun the playbook, and confirm the new content is served. Do you need to restart the container? Why or why not?

### Exercise 4 – Parameterize the port

Add variables `host_port` (default `81`) and `container_name` (default `nginx-site`) to the `vars:` section. Use them in the `ports:`, `wait_for`, and `site_url` settings so the port is defined in only one place. Re-run with an override:

```bash
ansible-playbook -i inventory.ini deploy_website.yaml -e host_port=8081
```

### Exercise 5 – Clean up

Write a second playbook, `cleanup.yaml`, that removes the container and the `/opt/website` directory.

```yaml
- name: Remove website deployment
  hosts: remotes
  become: yes
  tasks:
    - name: Remove NGINX container
      community.docker.docker_container:
        name: nginx-site
        state: absent

    - name: Remove website directory
      ansible.builtin.file:
        path: /opt/website
        state: absent
```

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Failed to import the required Python library (docker)` | Docker SDK missing on remote | `sudo apt install -y python3-docker` |
| `Cannot connect to the Docker daemon` | Docker not running | `sudo systemctl enable --now docker` |
| `couldn't resolve module/action 'community.docker...'` | Collection not installed | `ansible-galaxy collection install community.docker` |
| `Missing sudo password` | `become` needs a password | Add `--ask-become-pass` or configure passwordless sudo |
| `wait_for` times out on port 81 | Port already in use or container failed | `sudo ss -tlnp \| grep 81` and `sudo docker logs nginx-site` |
| Page unreachable from other hosts | UFW blocking 81 | `sudo ufw allow 81/tcp` |
| Page shows the NGINX default page | Volume path wrong | Check `sudo docker inspect nginx-site` mounts |
| `apt` lock errors when installing Docker | Another apt process running | Wait a minute and retry |

---

## 9. Deliverables

Submit:

1. The project folder (`inventory.ini`, `deploy_website.yaml`, `files/index.html`)
2. Screenshots or pasted output of:
   - The full playbook run
   - `docker ps` on the remote host
   - `curl http://localhost:81`
3. Written answers to Exercises 1 and 3
4. Your improved validation task from Exercise 2

---

## 10. Grading Rubric (100 points)

| Criteria | Points |
|---|---|
| Project set up correctly (inventory, files, collection installed) | 15 |
| Playbook runs successfully end to end | 25 |
| Container running with correct port mapping and volume | 15 |
| Site content verified on the remote host | 10 |
| Exercise 2: real content validation that fails on mismatch | 15 |
| Exercise 5: working cleanup playbook | 10 |
| Written answers and screenshots | 10 |

**Bonus (up to +10):** complete Exercise 4 with variables used consistently (+5), or add a task that installs Docker and the Python SDK through Ansible so the host needs no manual prep (+5).

---

## 11. Quick Reference

```bash
# Install collection
ansible-galaxy collection install community.docker

# Syntax check
ansible-playbook -i inventory.ini deploy_website.yaml --syntax-check

# Run
ansible-playbook -i inventory.ini deploy_website.yaml

# Run cleanup
ansible-playbook -i inventory.ini cleanup.yaml
```
