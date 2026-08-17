# Cloud-1

An Ansible playbook that automates the deployment of the [Inception](https://github.com/NouhailaElBrighli/inception) Docker stack (NGINX, WordPress, MariaDB) onto a remote cloud server. This is the Cloud-1 project from the 42 network curriculum.

Instead of SSHing into a server and running commands by hand, `site.yml` provisions Docker, ships the project files, and brings up each container — all in one run.

## What it does

1. **Installs Docker** on the target host (Docker Engine, CLI, containerd, and the Compose plugin) via the official Docker APT repository.
2. **Copies the Inception project** to the server and generates a `.env` file from a Jinja2 template, filling in the domain name and database/WordPress credentials.
3. **Creates the shared Docker network** and persistent data directories for MariaDB and WordPress.
4. **Builds and starts each service** — MariaDB, WordPress, then NGINX — as separate, independently taggable roles.

All secrets (DB passwords, WordPress admin/user passwords) are encrypted with **Ansible Vault** and never committed in plaintext.

## Project structure

```
.
├── ansible.cfg              # inventory, SSH user/key, privilege escalation
├── inventory.ini            # target host(s)
├── site.yml                 # main playbook — role order + final URL message
├── group_vars/all/
│   ├── vars.yml              # non-secret config (project paths, DB/user names)
│   └── vault.yml             # encrypted secrets (Vault)
└── roles/
    ├── docker/               # installs Docker + Compose plugin
    ├── common/                # shared prep: data dirs, copies project, templates .env, creates network
    ├── mariadb/               # docker compose up mariadb
    ├── wordpress/             # docker compose up wordpress
    └── nginx/                 # docker compose up nginx
```

## Requirements

- Ansible installed locally
- A remote Ubuntu server (tested with a `ubuntu` SSH user)
- An SSH keypair for the target server
- The `Inception` project directory available locally (the `common` role copies it to the remote host)

## Setup

1. Point `inventory.ini` at your server:
   ```ini
   [webservers]
   Cloud1 ansible_host=YOUR_SERVER_IP
   ```
2. Update `ansible.cfg` with the path to your SSH private key:
   ```ini
   private_key_file = ~/path/to/your-key.pem
   ```
3. Fill in `group_vars/all/vars.yml` with your domain, DB name/user, and WordPress admin details.
4. Encrypt your secrets with Vault:
   ```bash
   ansible-vault edit group_vars/all/vault.yml
   ```
   This should define `vault_mysql_password`, `vault_mysql_root_password`, `vault_wp_admin_password`, and `vault_wp_user_2_password`.

## Usage

Run the full deployment:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Run a single stage using tags (useful when iterating on one service without rebuilding everything):

```bash
ansible-playbook site.yml --tags docker      # install Docker only
ansible-playbook site.yml --tags mariadb     # (re)build MariaDB container only
ansible-playbook site.yml --tags wordpress   # (re)build WordPress container only
ansible-playbook site.yml --tags nginx       # (re)build NGINX container only
```

At the end of a successful run, the playbook prints the URL where the site is reachable.

## Notes

- The `common` role is shared prep work (data directories, project files, templated `.env`, Docker network) used by every service role, so it isn't duplicated three times.
- Passwords are never stored in plaintext — `vault.yml` is Vault-encrypted and only decrypted in memory when the `.env` template is rendered.
