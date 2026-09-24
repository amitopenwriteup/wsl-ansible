# Workshop: Simulate RDS MySQL Setup with Ansible (CentOS or Debian/Ubuntu)

## Prerequisites

Install collection:

```bash
ansible-galaxy collection install community.mysql --force
```

PyMySQL is installed by the playbook's first task using the `package` module, so it works on either OS family. Check it's installed:

```bash
ansible all -b -m command -a "python3 -c 'import pymysql'"
```

---

## Playbook (`mysql_setup.yml`)

```bash
vi mysql_setup.yml
```

```yaml
---
---
- name: Simulate RDS MySQL setup
  hosts: all
  become: yes
  vars:
    mysql_root_password: "RootPass123!"
    db_name: "testdb"
    db_user: "dbuser"
    db_user_password: "UserPass123!"
    mysql_socket: "{{ '/var/lib/mysql/mysql.sock' if ansible_os_family == 'RedHat' else '/run/mysqld/mysqld.sock' }}"

  tasks:
    - name: Ensure Python 3, pip, and PyMySQL are installed
      package:
        name:
          - python3
          - python3-pip
          - python3-pymysql
        state: present

    - name: Install MariaDB server
      package:
        name: mariadb-server
        state: present

    - name: Start and enable MariaDB service
      service:
        name: mariadb
        state: started
        enabled: true

    - name: Set MySQL root password
      community.mysql.mysql_user:
        name: root
        host_all: yes
        password: "{{ mysql_root_password }}"
        login_unix_socket: "{{ mysql_socket }}"
      ignore_errors: true

    - name: Create .my.cnf for root so future logins don't need -p
      template:
        src: my.cnf.j2
        dest: /root/.my.cnf
        owner: root
        group: root
        mode: "0600"

    - name: Create a MySQL database
      community.mysql.mysql_db:
        name: "{{ db_name }}"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
        login_unix_socket: "{{ mysql_socket }}"

    - name: Create a MySQL user with privileges
      community.mysql.mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_user_password }}"
        priv: "{{ db_name }}.*:ALL"
        host: "%"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
        login_unix_socket: "{{ mysql_socket }}"
```

---

## Template (`my.cnf.j2`)

```bash
vi my.cnf.j2
```

```ini
[client]
user=root
password={{ mysql_root_password }}
```

Lets root run `mysql` without typing `-p`.

---

## Run

```bash
ansible-playbook mysql_setup.yml --syntax-check
ansible-playbook mysql_setup.yml
```

---

## Verify

```bash
ansible all -b -m command -a "systemctl is-active mariadb"
ansible all -b -m shell -a "mysql -e 'SHOW DATABASES;'"
ansible all -b -m shell -a "mysql -e \"SELECT user,host FROM mysql.user;\""
```
