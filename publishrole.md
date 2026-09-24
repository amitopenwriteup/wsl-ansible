# Publish a Role to Ansible Galaxy

**Goal:** Push the role `test` created in Lab 4 to GitHub, then import it into Ansible Galaxy so anyone can install it.

**Before you start**

- The `test` role from Lab 4 (contains `tasks/`, `files/`, `meta/`, and so on)
- A GitHub account
- A Galaxy account (log in at [galaxy.ansible.com](https://galaxy.ansible.com) using GitHub)
- `git` installed on the control node

> **Naming tip:** Galaxy takes the role name from the **GitHub repo name**. A repo called `test` gives a role called `test`, which is too generic. In this guide we use the repo name `myansiblerole`.

---

## Step 1: Create a New Repo on GitHub

1. Log in to GitHub and click **New repository**.
2. Repository name: `myansiblerole`
3. Set it to **Public** (Galaxy can only import public repos).
4. Do **not** add a README, `.gitignore`, or license (the role already has its own files).
5. Click **Create repository**.

---

## Step 2: Add, Commit and Push

Run the commands from **inside the role folder**, so the role files sit at the root of the repo.

```bash
cd test

git init -b main
git add .
git commit -m "adding role"

git remote add origin https://github.com/<github-user-name>/myansiblerole.git
git push -u origin main
```

Example with a real user name:

```bash
git remote add origin https://github.com/amitopenwriteup/myansiblerole.git
```

> **Authentication:** GitHub no longer accepts account passwords for `git push`. When asked for a password, use a **Personal Access Token** (GitHub → Settings → Developer settings → Personal access tokens), or set up SSH keys.

Refresh the repo page on GitHub and confirm you see `tasks/`, `files/`, `meta/`, `defaults/`, and the other role folders at the top level.

### Recommended: fill in `meta/main.yml` before pushing

Galaxy shows this information on the role page.

```bash
vi meta/main.yml
```

```yaml
galaxy_info:
  author: your_name
  description: Installs Apache httpd and copies an index.html file
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: EL
      versions: [all]
  galaxy_tags:
    - apache
    - httpd

dependencies: []
```

Then commit again:

```bash
git add .
git commit -m "update role metadata"
git push
```

---

## Step 3: Get Your API Token

1. Go to the Galaxy home page: [galaxy.ansible.com](https://galaxy.ansible.com).
2. Log in.
3. Open **Collections** → **API token**.
4. Click **Load token** and copy it.

> **Keep it secret.** Never commit the token to Git or paste it into shared documents. If it leaks, reset it on the same page.

---

## Step 4: Import the Role into Galaxy

Syntax:

```bash
ansible-galaxy import <GitHub user name> <repo name on github> --token <token id>
```

Example:

```bash
ansible-galaxy import amitopenwriteup myansiblerole --token <tokenid>
```

### Sample output

```text
===== CLONING REPO =====
cloning https://github.com/amitopenwriteup/myansiblerole ...

===== GIT ATTRIBUTES =====
github_reference(branch): main
github_commit: a09320ab54ec444963bd6ab9f9243f8966908
github_commit_message: adding role
github_commit_date: 2024-11-07T13:58:07+00:00

===== LOADING ROLE =====
Importing with galaxy-importer 0.4.20
Determined role name to be myansiblerole
Linting role myansiblerole via ansible-lint...
myansiblerole/tests/test.yml:5:7: syntax-check[specific]: the role 'test' was not found in /tmp/tmp2gub527c/myansiblerole/tests/roles:/app/.cache/ansible-compat/bcd9b5/roles:/app/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles:/tmp/tmp2gub527c/myansiblerole/tests
...ansible-lint run complete
Legacy role loading complete

===== PROCESSING LOADER RESULTS ====
enumerated role name myansiblerole
created new role id:39534 amitopenwriteup.myansiblerole

===== COMPUTING ROLE VERSIONS ====

==== SAVING ROLE ====

Import completed
```

### What the output means

| Section | Meaning |
|---|---|
| `CLONING REPO` | Galaxy clones your public GitHub repo |
| `GIT ATTRIBUTES` | Branch, commit ID, message, and date that were imported |
| `LOADING ROLE` | Galaxy detects the role name from the repo name and lints it with `ansible-lint` |
| `PROCESSING LOADER RESULTS` | Creates the role entry, here `amitopenwriteup.myansiblerole` (role id `39534`) |
| `Import completed` | The role is now published |

### About the lint message

```text
myansiblerole/tests/test.yml:5:7: syntax-check[specific]: the role 'test' was not found ...
```

This is a **warning, not a failure**, and the import still completes. It happens because the generated `tests/test.yml` calls a role named `test`, but the repo (and role) is now named `myansiblerole`.

To clear it, either delete the tests folder:

```bash
git rm -r tests
git commit -m "remove generated tests folder"
git push
```

or edit `tests/test.yml` and change `- test` to `- myansiblerole`.

---

## Step 5: Verify on Galaxy

1. Open [galaxy.ansible.com](https://galaxy.ansible.com) → **Legacy Roles** (or search `amitopenwriteup.myansiblerole`).
2. Confirm the role page shows your description, author, and version info.

Install and use it from any machine:

```bash
ansible-galaxy role install amitopenwriteup.myansiblerole
```

```yaml
---
- hosts: all
  become: true
  roles:
    - amitopenwriteup.myansiblerole
```

---

## Updating the Role Later

Push changes to GitHub, then re-run the import:

```bash
git add .
git commit -m "describe your change"
git push

ansible-galaxy import amitopenwriteup myansiblerole --token <tokenid>
```

To publish a numbered version, tag the commit before importing:

```bash
git tag v1.0.0
git push origin v1.0.0
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `git push` asks for a password and fails | GitHub disabled password login | Use a Personal Access Token or SSH key |
| `Repository not found` on import | Repo is private, or the user/repo name is misspelled | Make the repo public and check the spelling (it is case-sensitive) |
| Role name is `test` on Galaxy | Repo was named `test` | Rename the GitHub repo, then re-import |
| `Invalid token` / `401` | Wrong or reset token | Copy a fresh token from the Galaxy API token page |
| Role page has no description | `meta/main.yml` is still the default | Fill in `galaxy_info`, push, and re-import |
| Files sit inside a `test/` subfolder on GitHub | `git init` was run in the parent folder | Run `git init` inside the role folder so files are at the repo root |

---

## Quick Recap

1. Create a public GitHub repo.
2. `git init`, `add`, `commit`, and `push` from **inside** the role folder.
3. Get the API token from Galaxy (**Collections → API token**).
4. Run `ansible-galaxy import <user> <repo> --token <token>`.
5. Install with `ansible-galaxy role install <user>.<repo>`.
