# Cloud-1: Automated WordPress Deployment with Ansible

Cloud-1 uses **Ansible** to deploy a full, containerized **WordPress** stack onto a remote Ubuntu server, such as an Azure VM. Running one command takes a fresh machine to a working website:

- **Docker Engine** and the **Docker Compose** plugin are installed
- a **UFW firewall** is set up so only SSH, HTTP and HTTPS are reachable
- an optional **Let's Encrypt TLS certificate** is obtained
- **MySQL**, **WordPress (PHP-FPM)**, **phpMyAdmin** and **Nginx** start as Docker containers
- WordPress is **installed automatically** through WP-CLI, so no setup wizard is needed

The playbook is **idempotent**: you can run it again at any time, and it only changes what needs changing.

---

## Table of contents

1. [Architecture](#architecture)
2. [Repository layout](#repository-layout)
3. [Prerequisites](#prerequisites)
4. [Configuration](#configuration)
5. [Deploying](#deploying)
6. [What each role does](#what-each-role-does)
7. [Accessing the services](#accessing-the-services)
8. [Day-to-day operations](#day-to-day-operations)
9. [Security notes](#security-notes)
10. [Troubleshooting](#troubleshooting)
11. [Known limitations](#known-limitations)

---

## Architecture

```
                        Internet
                           │
              ┌────────────┴────────────┐
              │   UFW: 22, 80, 443 only │        Remote Ubuntu host
              └────────────┬────────────┘
                           │ :80 / :443
                  ┌────────▼────────┐
                  │      nginx      │  nginx:alpine
                  │ (reverse proxy, │  - serves static files (read-only volume)
                  │  TLS endpoint)  │  - FastCGI → wordpress:9000
                  └───┬─────────┬───┘  - /phpmyadmin/ → phpmyadmin:80
       frontend net   │         │
             ┌────────▼───┐ ┌───▼──────────┐
             │ wordpress  │ │  phpmyadmin  │
             │ php8.2-fpm │ │     5.2      │
             └────────┬───┘ └───┬──────────┘
       backend net    │         │      (internal: no internet access)
       (internal)   ┌─▼─────────▼─┐
                    │    mysql    │  mysql:8.0
                    └─────────────┘

   wpcli (wordpress:cli-php8.2): a one-off container under the "tools" profile,
   started only when Ansible runs WP-CLI commands.
```

### Docker networks

| Network    | Members                              | Purpose |
|------------|--------------------------------------|---------|
| `frontend` | nginx, wordpress, phpmyadmin         | Lets Nginx reach the PHP and phpMyAdmin backends |
| `backend`  | mysql, wordpress, phpmyadmin, wpcli  | Database traffic. It is marked `internal: true`, so it has **no outside access** |

MySQL sits only on the internal `backend` network, so it can **never be reached from outside**. Nginx is the only container that publishes ports to the host.

### Docker volumes

| Volume           | Mounted in                                   | Contents |
|------------------|----------------------------------------------|----------|
| `mysql_data`     | mysql (`/var/lib/mysql`)                     | Database files |
| `wordpress_data` | wordpress, wpcli (read-write); nginx (read-only) | WordPress core, themes, plugins, uploads |

The data survives container restarts, recreations and image upgrades.

---

## Repository layout

```
cloud-1/
└── ansible/
    ├── ansible.cfg                  # Points Ansible at inventory.ini
    ├── inventory.ini                # Target host(s) and per-host settings
    ├── site.yml                     # Main playbook: applies the 4 roles in order
    ├── group_vars/
    │   └── all/
    │       ├── vars.yaml            # Non-secret settings (DB name, WP title, ...)
    │       └── vault.yml            # Secrets, encrypted with Ansible Vault
    └── roles/
        ├── docker/
        │   └── tasks/main.yml       # Installs Docker Engine + Compose plugin
        ├── firewall/
        │   └── tasks/main.yml       # Configures UFW
        ├── tls/
        │   └── tasks/main.yml       # Gets a Let's Encrypt certificate (optional)
        └── wordpress/
            ├── tasks/main.yml       # Renders configs, starts the stack, installs WP
            ├── handlers/main.yml    # Restarts Nginx when its config changes
            └── templates/
                ├── docker-compose.yml.j2
                └── nginx.conf.j2
```

---

## Prerequisites

### On your machine (the control node)

- **Ansible** 2.12 or newer (`pip install ansible` or your OS package manager)
- An **SSH private key** that can log in to the target server
- The **Ansible Vault password** for `group_vars/all/vault.yml`

> Ansible does not run natively on Windows. On Windows, use WSL or a Linux VM.

### On the target server

- **Ubuntu** (the Docker role uses Docker's Ubuntu apt repository)
- A user that can log in over SSH and has `sudo` rights (the playbook runs with `become: true`)
- Python 3, which Ubuntu includes by default
- **For HTTPS:** a **domain name** whose DNS A record points to the server's public IP (a free DuckDNS subdomain works), and inbound **port 80 open** in the cloud provider's firewall or network security group, so Let's Encrypt can validate the domain
- Inbound ports **22, 80 and 443** allowed in the cloud provider's firewall (on Azure, the VM's Network Security Group)

---

## Configuration

### 1. Inventory: `ansible/inventory.ini`

The inventory defines the `webserver` group that the playbook targets. Each host line holds the connection details and per-host settings:

```ini
[webserver]
cloud1-azure ansible_host=<SERVER_PUBLIC_IP> ansible_user=<SSH_USER> ansible_ssh_private_key_file=<PATH_TO_PRIVATE_KEY> domain_name=<your.domain.example> use_letsencrypt=True
```

| Variable                       | Meaning |
|--------------------------------|---------|
| `ansible_host`                 | Public IP address of the server |
| `ansible_user`                 | SSH user (for example `azureuser` on Azure) |
| `ansible_ssh_private_key_file` | Path to the SSH private key **on your machine** |
| `domain_name`                  | Domain the site is served on. It is used in the Nginx config, the WordPress URL and the TLS certificate |
| `use_letsencrypt`              | `True` for HTTPS with Let's Encrypt, `False` for plain HTTP |

To deploy to **several servers**, add one line per host under `[webserver]`.

> **Variable precedence:** `vars.yaml` sets `use_letsencrypt: false` as the default. A value set on the host line in `inventory.ini` takes precedence, so you can turn HTTPS on or off for each host.

### 2. General settings: `ansible/group_vars/all/vars.yaml`

| Variable                   | Default        | Meaning |
|----------------------------|----------------|---------|
| `app_dir`                  | `/opt/cloud1`  | Directory on the server for `docker-compose.yml`, `nginx.conf` and `certs/` |
| `use_letsencrypt`          | `false`        | Default TLS mode (the inventory can override it) |
| `site_scheme`              | derived        | `https` when `use_letsencrypt` is true, otherwise `http` |
| `letsencrypt_email`        | (set)          | Email for Let's Encrypt account and expiry notices |
| `mysql_database`           | `wordpress`    | Database name |
| `mysql_user`               | `wp_user`      | Database user for WordPress |
| `wordpress_title`          | `Cloud-1`      | Site title |
| `wordpress_admin_user`     | `cloud1_admin` | WordPress administrator login |
| `wordpress_admin_email`    | (set)          | WordPress administrator email |

The password variables in this file point to vault variables, so no secret is stored in plain text:

```yaml
mysql_root_password:      "{{ vault_mysql_root_password }}"
mysql_password:           "{{ vault_mysql_password }}"
wordpress_admin_password: "{{ vault_wordpress_admin_password }}"
```

### 3. Secrets: `ansible/group_vars/all/vault.yml`

This file is encrypted with **Ansible Vault** and must define these three variables:

```yaml
vault_mysql_root_password: "..."
vault_mysql_password: "..."
vault_wordpress_admin_password: "..."
```

Common vault commands (run them from `ansible/`):

```bash
ansible-vault view   group_vars/all/vault.yml   # read the secrets
ansible-vault edit   group_vars/all/vault.yml   # change the secrets
ansible-vault rekey  group_vars/all/vault.yml   # change the vault password
```

To start over with your own secrets:

```bash
ansible-vault create group_vars/all/vault.yml
```

---

## Deploying

Run every command from the `ansible/` directory, so that `ansible.cfg` and the inventory are picked up:

```bash
cd ansible

# 1. Check that Ansible can reach the server
ansible webserver -m ping

# 2. Optional: preview the changes without applying them
ansible-playbook site.yml --ask-vault-pass --check --diff

# 3. Deploy
ansible-playbook site.yml --ask-vault-pass
```

To skip the password prompt, keep the vault password in a file that is **not committed to git**:

```bash
echo 'your-vault-password' > ~/.cloud1_vault_pass && chmod 600 ~/.cloud1_vault_pass
ansible-playbook site.yml --vault-password-file ~/.cloud1_vault_pass
```

A first run on a fresh server takes a few minutes, mostly for package installation and image downloads. Later runs are much faster and report few or no changes.

---

## What each role does

`site.yml` applies four roles in this order, with root privileges, to every host in `webserver`:

### 1. `docker`: install the container runtime

1. Installs `ca-certificates` and `curl`.
2. Downloads Docker's official GPG key to `/etc/apt/keyrings/docker.asc`.
3. Detects the CPU architecture (`dpkg --print-architecture`).
4. Removes any old one-line `docker.list` repository file and writes a modern deb822 `docker.sources` file for the matching Ubuntu release.
5. Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin` and `docker-compose-plugin`.
6. Makes sure the Docker service is **started and enabled at boot**.

### 2. `firewall`: lock down the host

1. Installs **UFW**.
2. Sets the default policy to **deny all incoming** and **allow all outgoing** traffic.
3. Opens **22/tcp** (SSH), **80/tcp** (HTTP) and **443/tcp** (HTTPS). A task reports "changed" only when it actually adds a rule.
4. Enables UFW without asking for confirmation.

### 3. `tls`: get an HTTPS certificate (only when `use_letsencrypt` is true)

1. Creates `{{ app_dir }}` and `{{ app_dir }}/certs`.
2. Installs **Certbot**.
3. Checks whether a certificate already exists in `/etc/letsencrypt/live/<domain>/`.
4. If there is **no certificate** but the stack is already deployed, it **stops the Nginx container** to free port 80.
5. Runs `certbot certonly --standalone`. Certbot starts its own temporary web server on port 80 to answer the HTTP-01 challenge. Standalone mode is used because Nginx is not running yet at this point.
6. Copies `fullchain.pem` (mode `0644`) and `privkey.pem` (mode `0600`) into `{{ app_dir }}/certs/`, which is mounted read-only into the Nginx container.

When a certificate already exists, Certbot is skipped. Only the copy steps run, which refreshes the files if Certbot renewed them.

### 4. `wordpress`: deploy and install the application

1. Creates `{{ app_dir }}`.
2. Renders `docker-compose.yml` from its template with mode `0600`, because it contains the database passwords. Diff output is turned off so the secrets never show up in logs.
3. Renders `nginx.conf`. If the file changes, a **handler restarts the Nginx container**.
4. Runs `docker compose up -d` to start mysql, wordpress, phpmyadmin and nginx.
5. **Waits for MySQL**: it retries `mysqladmin ping` up to 30 times, 2 seconds apart.
6. **Waits for WordPress**: it retries until the image's entrypoint has generated `wp-config.php`.
7. Runs `wp core is-installed` in a throwaway `wpcli` container.
8. **Only if WordPress is not installed yet**, runs `wp core install` with the URL, title and admin account from the variables. This task uses `no_log: true` so the admin password is never printed.

### The Nginx configuration (`nginx.conf.j2`)

- **HTTPS mode:** one server block on port 80 sends a `301` redirect to HTTPS. The main block listens on `443 ssl` with the Let's Encrypt certificate.
- **HTTP mode:** a single server block listens on port 80.
- `/` uses `try_files $uri $uri/ /index.php?$args`, which WordPress needs for pretty permalinks.
- `*.php` requests go over **FastCGI** to `wordpress:9000`. `HTTPS` is passed along, so WordPress knows when a request came in over TLS.
- `/phpmyadmin` redirects to `/phpmyadmin/`, which is **reverse-proxied** to the phpMyAdmin container with `X-Forwarded-Proto` set.
- Nginx serves static files (CSS, JS, images, uploads) directly from the shared `wordpress_data` volume, which it mounts **read-only**.

---

## Accessing the services

The scheme is `https` when Let's Encrypt is on, otherwise `http`.

| Service          | URL                                     | Credentials |
|------------------|-----------------------------------------|-------------|
| WordPress site   | `https://<domain_name>/`                | none |
| WordPress admin  | `https://<domain_name>/wp-admin/`       | `wordpress_admin_user` and `vault_wordpress_admin_password` |
| phpMyAdmin       | `https://<domain_name>/phpmyadmin/`     | `mysql_user` and `vault_mysql_password` (or `root` and `vault_mysql_root_password`) |

---

## Day-to-day operations

SSH into the server, then work from the application directory:

```bash
cd /opt/cloud1

sudo docker compose ps                     # container status
sudo docker compose logs -f nginx          # follow the logs of one service
sudo docker compose restart wordpress      # restart one service
sudo docker compose down                   # stop everything (data is kept in volumes)
sudo docker compose up -d                  # start everything again
```

### Running WP-CLI commands

```bash
sudo docker compose --profile tools run --rm wpcli wp plugin list
sudo docker compose --profile tools run --rm wpcli wp user list
sudo docker compose --profile tools run --rm wpcli wp core update
```

### Backups

```bash
# Database dump
sudo docker compose exec -T mysql sh -c \
  'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE"' > backup.sql

# WordPress files (themes, plugins, uploads)
sudo docker run --rm -v cloud1_wordpress_data:/data -v "$PWD":/backup alpine \
  tar czf /backup/wordpress_data.tgz -C /data .
```

> Compose prefixes volume names with the project name, which defaults to the directory name. With `app_dir=/opt/cloud1`, the volumes are `cloud1_mysql_data` and `cloud1_wordpress_data`. Check with `sudo docker volume ls`.

### Applying configuration changes

After you edit variables or templates, run the playbook again:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Only the affected parts change. For example, an Nginx config change re-renders `nginx.conf` and restarts the Nginx container.

### Resetting everything (destructive)

This **deletes the database and all WordPress content**:

```bash
cd /opt/cloud1 && sudo docker compose down -v
```

Running the playbook afterwards rebuilds a fresh site.

---

## Security notes

- **Secrets** live only in the vault-encrypted `vault.yml`. Never commit the vault password.
- `docker-compose.yml` on the server holds the passwords, so it is written with mode `0600` (readable by root only).
- The WordPress install task uses `no_log: true`, and the Compose template is rendered with `diff: false`, so passwords never appear in Ansible output.
- **MySQL is never exposed**. It runs only on an internal Docker network.
- The **firewall** allows only ports 22, 80 and 443.
- The TLS private key is stored with mode `0600` and mounted **read-only** into Nginx.
- Nginx mounts the WordPress files **read-only**, so a compromised web server cannot change them.
- **phpMyAdmin is publicly reachable** at `/phpmyadmin/`. For a production site, consider restricting it by IP, adding HTTP basic auth in Nginx, or removing it.
- The inventory and `vars.yaml` contain real infrastructure details (IP address, SSH key path, emails). Keep the repository private, or move those values out of version control.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| `UNREACHABLE` / SSH errors | Check `ansible_host`, `ansible_user` and the key path in `inventory.ini`, and make sure port 22 is open in the cloud firewall. Test with `ssh -i <key> <user>@<ip>`. |
| `Attempting to decrypt but no vault secrets found` | Add `--ask-vault-pass` or `--vault-password-file`. |
| Certbot fails with a challenge or connection error | The domain's DNS must point at the server, and port 80 must be open in **both** UFW and the cloud provider's firewall or NSG. Something else on port 80 (such as a running Nginx container) also blocks it. |
| "Wait until MySQL accepts connections" times out | Run `sudo docker compose logs mysql`. If you changed DB passwords after the first deployment, note that MySQL keeps the original credentials stored in its volume. |
| Site redirects to the wrong URL or scheme | WordPress saves the site URL at install time. If you switch between HTTP and HTTPS or change the domain later, update it with `wp option update home <url>` and `wp option update siteurl <url>` through WP-CLI. |
| 502 Bad Gateway | The `wordpress` or `phpmyadmin` container is down. Check `sudo docker compose ps` and the container logs. |

---

## Known limitations

- **Certificate renewal is not automated.** Certbot's renewal timer updates `/etc/letsencrypt/live/...`, but standalone renewal needs port 80, which Nginx holds, and the renewed files only reach `{{ app_dir }}/certs/` when the playbook runs again. Before the certificate expires (Let's Encrypt certificates last 90 days), stop Nginx, run `certbot renew`, and run the playbook again. You could also switch to webroot mode with a renewal hook.
- **Ubuntu only.** The Docker role uses Docker's Ubuntu repository.
- **Single node.** Each host runs its own independent stack with its own database. There is no shared storage or load balancing between hosts.
- **Image tags float** within their major line (`mysql:8.0`, `nginx:alpine`, ...), so redeploying can pull newer patch releases.
