# Ansible Workshop: Loops, Conditionals, Handlers & More

A hands-on reference for four of the most-used building blocks in Ansible playbooks: `loop`, `when`, `handlers`, and a grab-bag of other essentials you'll reach for constantly.

---

## 1. `loop` — Repeating a Task

Use `loop` whenever you'd otherwise copy-paste the same task with only one value changed (installing several packages, creating several users, looping over a list of files).

### Basic loop

```yaml
- name: Install a list of packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl
```

Each pass through the loop, the current value is available as `item`.

### Looping over a variable

```yaml
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

### Looping over a list of dictionaries

```yaml
- name: Create users
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: "alice", groups: "sudo" }
    - { name: "bob", groups: "developers" }
```

### Naming the loop variable (`loop_control`)

Nested loops or long loop bodies get confusing with a generic `item`. Rename it:

```yaml
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

### Old-style `with_*` loops

You'll see `with_items`, `with_dict`, `with_fileglob`, etc. in older playbooks. They still work, but `loop` (plus filters like `flatten`, `dict2items`) is the modern, recommended approach.

> **Workshop exercise:** Write a task that uses `loop` to create three directories: `/opt/app/logs`, `/opt/app/data`, `/opt/app/config`.

---

## 2. `when` — Conditional Execution

`when` skips or runs a task based on a condition. It's evaluated **before** the task runs, once per host (and once per loop item, if combined with `loop`).

### Basic condition

```yaml
- name: Restart service only on Debian-family hosts
  ansible.builtin.service:
    name: nginx
    state: restarted
  when: ansible_facts['os_family'] == "Debian"
```

### Combining conditions

```yaml
- name: Only run on CentOS 8+ web servers
  ansible.builtin.debug:
    msg: "Matched!"
  when:
    - ansible_facts['distribution'] == "CentOS"
    - ansible_facts['distribution_major_version'] | int >= 8
    - "'webservers' in group_names"
```

A YAML list under `when` is an implicit **AND**. For **OR**, use `or` inline:

```yaml
when: ansible_facts['distribution'] == "CentOS" or ansible_facts['distribution'] == "RedHat"
```

### `when` with a registered variable

```yaml
- name: Check if a file exists
  ansible.builtin.stat:
    path: /etc/myapp.conf
  register: config_file

- name: Only run if the file exists
  ansible.builtin.debug:
    msg: "Config found!"
  when: config_file.stat.exists
```

### `when` with a loop

`when` is checked separately for every item:

```yaml
- name: Install only the packages not already listed
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop: "{{ packages }}"
  when: item != "curl"
```

> **Common pitfall:** Don't wrap the condition in `{{ }}` — write `when: x == "y"`, not `when: "{{ x == 'y' }}"`.

---

## 3. Handlers — Run Tasks Only on Change, Only Once

A **handler** is a task that only runs when notified — and Ansible runs it **once**, at the end of the play, no matter how many tasks notify it.

### Defining and notifying a handler

```yaml
tasks:
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

Both `copy` tasks notify the same handler, but **it only fires once**, and only if at least one of them actually changed something.

### Key rules

- A handler fires only when the notifying task reports `changed`. A task that reports `ok` (nothing to do) never triggers it.
- Handlers normally run **at the end of the play**, after all tasks — not immediately when notified.
- Handler names must be unique within the scope where they're notified (or use `listen`, below).

### Forcing handlers to run early

```yaml
- name: Flush handlers now instead of at the end of the play
  ansible.builtin.meta: flush_handlers
```

### `listen` — one notify, many handlers

```yaml
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

tasks:
  - name: Deploy new config
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: "restart web stack"
```

Both handlers fire, in the order they're defined, because they both `listen` for the same topic.

> **Workshop exercise:** Add a handler called `Reload systemd` that runs `systemctl daemon-reload`, and have a task that copies a `.service` file notify it.

---

## 4. Other Essentials Worth Knowing

A few more directives you'll use in almost every real playbook.

### `register` — capture a task's result

```yaml
- name: Check disk usage
  ansible.builtin.command: df -h /
  register: disk_usage

- name: Show the result
  ansible.builtin.debug:
    var: disk_usage.stdout
```

### `block` / `rescue` / `always` — group and handle errors

```yaml
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

`block` also lets you apply one `when` to a whole group of tasks at once, instead of repeating it on each task.

### `tags` — run only part of a playbook

```yaml
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

```bash
ansible-playbook site.yml --tags config
ansible-playbook site.yml --skip-tags install
```

### `vars`, `vars_files`, and precedence

```yaml
vars:
  app_port: 8080

vars_files:
  - secrets.yml
```

Ansible has a well-defined [variable precedence order](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable) — role defaults lose to play vars, which lose to `-e` extra-vars on the command line, roughly from "easiest to override" to "wins no matter what."

### Facts — information Ansible gathers automatically

```yaml
- name: Show the OS family
  ansible.builtin.debug:
    var: ansible_facts['os_family']
```

Facts power most `when` conditions you'll write. Turn gathering off with `gather_facts: false` on plays that don't need it, to speed up runs.

---

## Quick Reference Table

| Directive | Purpose | Fires |
|---|---|---|
| `loop` | Repeat a task over a list | Once per item |
| `when` | Conditionally run a task | Evaluated before each run (and each loop item) |
| `notify` + `handlers` | Run cleanup/restart tasks only on change | Once, at end of play (or on `flush_handlers`) |
| `listen` | Let multiple handlers respond to one notify | Same as handlers |
| `register` | Save a task's result for later use | N/A — just storage |
| `block/rescue/always` | Group tasks, handle errors, share `when` | Like try/except/finally |
| `tags` | Run/skip subsets of a playbook from the CLI | On demand |

---

## Suggested Workshop Flow

1. **Warm-up:** Write a playbook that loops over 3 packages and installs them.
2. **Add conditionals:** Make one package install only on Debian-family hosts using `when`.
3. **Add a handler:** Notify a handler to restart a service when a config file changes; demonstrate it firing only once even with two notifying tasks.
4. **Break it on purpose:** Have someone add a task that fails, then wrap it in `block/rescue/always` and show `rescue` catching it.
5. **Wrap-up:** Use `register` + `debug` to inspect a command's output live.
