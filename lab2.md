---
# =============================================================================
# Ansible Workshop Labs — all examples converted to full-playbook format
# Run any lab with:
#   ansible-playbook -i inventory.ini labloop.yaml --tags <lab-tag>
# Or run everything at once:
#   ansible-playbook -i inventory.ini labloop.yaml
# =============================================================================

# -----------------------------------------------------------------------------
# LAB 1 — Basic loop: install a list of packages
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 2 — Loop over a variable
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 3 — Loop over a list of dictionaries
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 4 — Naming the loop variable (loop_control)
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 5 — Workshop exercise: loop to create three directories
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 6 — when: basic condition
# -----------------------------------------------------------------------------
- name: test
  hosts: all
  tasks:
    - name: Restart service only on Debian-family hosts
      ansible.builtin.service:
        name: nginx
        state: restarted
      when: ansible_facts['os_family'] == "Debian"
      tags: lab6_when_basic

# -----------------------------------------------------------------------------
# LAB 7 — when: combining conditions (implicit AND)
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 8 — when: OR condition
# -----------------------------------------------------------------------------
- name: test
  hosts: all
  tasks:
    - name: Run on CentOS or RedHat
      ansible.builtin.debug:
        msg: "Matched CentOS or RedHat!"
      when: ansible_facts['distribution'] == "CentOS" or ansible_facts['distribution'] == "RedHat"
      tags: lab8_when_or

# -----------------------------------------------------------------------------
# LAB 9 — when: with a registered variable
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 10 — when: combined with a loop
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 11 — Handlers: notify runs once even with two notifying tasks
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 12 — Handlers: flush_handlers to force early run
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 13 — Handlers: listen (one notify, many handlers)
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 14 — Workshop exercise: Reload systemd handler
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 15 — register: capture a task's result
# -----------------------------------------------------------------------------
- name: test
  hosts: all
  tasks:
    - name: Check disk usage
      ansible.builtin.command: df -h /
      register: disk_usage
      tags: lab15_register
      changed_when: false

    - name: Show the result
      ansible.builtin.debug:
        var: disk_usage.stdout
      tags: lab15_register

# -----------------------------------------------------------------------------
# LAB 16 — block / rescue / always
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 17 — tags: run only part of a playbook
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 18 — vars, vars_files, and precedence
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 19 — Facts: information Ansible gathers automatically
# -----------------------------------------------------------------------------
- name: test
  hosts: all
  tasks:
    - name: Show the OS family
      ansible.builtin.debug:
        var: ansible_facts['os_family']
      tags: lab19_facts

# =============================================================================
# LAB 2 — Define Variables
# =============================================================================

# -----------------------------------------------------------------------------
# LAB 20 — Method 1: variables defined in the playbook (vars)
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 21 — Method 2: variables defined in inventory (host_vars)
# Requires inventory.ini to define, e.g.:
#   [local]
#   localhost ansible_connection=local ansible_user=amit app_name=myapp app_port=8080
# -----------------------------------------------------------------------------
- name: test
  hosts: all
  tasks:
    - name: Debug variables
      ansible.builtin.debug:
        msg: "App {{ app_name }} on port {{ app_port }}"
      tags: lab21_vars_inventory

# -----------------------------------------------------------------------------
# LAB 22 — Method 3: external variable file (group_vars)
# Requires group_vars/all.yml (or group_vars/<group>.yml) with:
#   package_name: nginx
#   service_name: nginx
#   port: 80
#   admin_email: admin@example.com
# -----------------------------------------------------------------------------
- name: test
  hosts: all
  tasks:
    - name: Install and start service
      ansible.builtin.apt:
        name: "{{ package_name }}"
        state: present
      tags: lab22_group_vars

# -----------------------------------------------------------------------------
# LAB 23 — Method 4: command-line variables (-e)
# Run with:
#   ansible-playbook -i inventory.ini labloop.yaml --tags lab23_cli_vars -e "package_name=nginx port=8080"
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 24 — Method 5: vars_files
# Requires vars.yml with:
#   package_name: apache2
#   service_name: apache2
#   port: 80
#   max_connections: 1000
#   enable_ssl: true
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 25 — Complete lab exercise: vars_files + vars + loop + service
# Requires vars.yml with:
#   packages:
#     - apache2
#     - curl
#     - wget
#   service_name: apache2
#   port: 80
#   state: present
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 26 — Variable types: string, list, dictionary
# -----------------------------------------------------------------------------
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

# -----------------------------------------------------------------------------
# LAB 27 — Debugging variables (all vars, specific var, custom message)
# -----------------------------------------------------------------------------
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
