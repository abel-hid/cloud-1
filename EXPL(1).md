# Cloud-1 — Explanation of the Final Ansible Code

This document explains **the current code inside `ansible.zip` exactly as it is now**.

The goal is not to add new features. The goal is to understand what every important part of the final project does and how the pieces connect.

The current project has two operating modes:

```text
use_letsencrypt=True
        ↓
Domain name + Let's Encrypt
        ↓
HTTP :80 redirects to HTTPS
HTTPS :443 serves the site
```

and:

```text
use_letsencrypt=False
        ↓
No TLS tasks
        ↓
HTTP :80 serves the site directly
No HTTPS listener from this Compose template
```

There is **no self-signed certificate path** in the current code.

---

# 1. The complete mental picture

There are two machines involved.

```text
LOCAL MACHINE
┌──────────────────────────────────────────────┐
│                                              │
│ ansible/                                     │
│ ├── ansible.cfg                              │
│ ├── inventory.ini                            │
│ ├── site.yml                                 │
│ ├── group_vars/                              │
│ └── roles/                                   │
│                                              │
│ Ansible runs here                            │
└──────────────────────┬───────────────────────┘
                       │
                       │ SSH :22
                       ▼
REMOTE UBUNTU SERVER / AZURE VM
┌──────────────────────────────────────────────┐
│                                              │
│ Docker                                       │
│                                              │
│ Internet                                     │
│    │                                         │
│    ▼                                         │
│ nginx                                        │
│  ├── WordPress PHP-FPM                       │
│  └── phpMyAdmin                              │
│         │                                    │
│         ▼                                    │
│       MySQL                                  │
│                                              │
└──────────────────────────────────────────────┘
```

You do not manually install the stack one command at a time on the server.

You run Ansible locally, and Ansible connects to the remote machine through SSH and brings it to the desired state.

---

# 2. Final project structure

The final archive contains:

```text
ansible/
├── ansible.cfg
├── inventory.ini
├── site.yml
│
├── group_vars/
│   └── all/
│       ├── vars.yaml
│       └── vault.yml
│
└── roles/
    ├── docker/
    │   └── tasks/
    │       └── main.yml
    │
    ├── firewall/
    │   └── tasks/
    │       └── main.yml
    │
    ├── tls/
    │   └── tasks/
    │       └── main.yml
    │
    └── wordpress/
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        └── templates/
            ├── docker-compose.yml.j2
            └── nginx.conf.j2
```

Think of the files this way:

```text
ansible.cfg
    ↓
Ansible configuration

inventory.ini
    ↓
Which remote machine to control and how to reach it

group_vars/all/vars.yaml
    ↓
Normal project variables

group_vars/all/vault.yml
    ↓
Encrypted secrets

site.yml
    ↓
Main playbook / entry point

roles/
    ↓
Actual infrastructure tasks

templates/
    ↓
Files that Ansible renders on the remote server
```

---

# 3. `ansible.cfg`

The final file is:

```ini
[defaults]
inventory = ./inventory.ini
```

`[defaults]` means these are default Ansible settings.

This line:

```ini
inventory = ./inventory.ini
```

means:

> When Ansible needs an inventory, use `inventory.ini` from this directory.

Because of this, when you are inside the `ansible/` directory, you can run:

```bash
ansible webserver -m ping
```

instead of always writing:

```bash
ansible -i inventory.ini webserver -m ping
```

So the connection is:

```text
ansible command
      ↓
ansible.cfg
      ↓
inventory = ./inventory.ini
      ↓
inventory.ini is loaded
```

---

# 4. `inventory.ini`

The final inventory contains:

```ini
[webserver]
cloud1-azure ansible_host=74.248.37.10 ansible_user=azureuser ansible_ssh_private_key_file=/home/bel-idri/Desktop/id_rsa domain_name=abel-hid.duckdns.org use_letsencrypt=True
```

We will break every part down.

---

## 4.1 `[webserver]`

```ini
[webserver]
```

This creates an Ansible host group named:

```text
webserver
```

Your main playbook later says:

```yaml
hosts: webserver
```

So Ansible takes all hosts inside this group and runs the play against them.

Conceptually:

```text
site.yml
hosts: webserver
       ↓
inventory.ini
[webserver]
       ↓
cloud1-azure
```

If you later put multiple hosts under `[webserver]`, Ansible can target all of them.

---

## 4.2 `cloud1-azure`

```ini
cloud1-azure
```

This is the **inventory hostname**.

It is the friendly name Ansible uses for this machine.

That is why Ansible output looks like:

```text
ok: [cloud1-azure]
```

It does not need to be the actual Linux hostname.

---

## 4.3 `ansible_host`

```ini
ansible_host=74.248.37.10
```

This is the real address Ansible connects to.

So:

```text
cloud1-azure
      ↓
74.248.37.10
```

Conceptually, SSH will connect to this IP.

---

## 4.4 `ansible_user`

```ini
ansible_user=azureuser
```

This means Ansible connects as the remote Linux user:

```text
azureuser
```

Conceptually:

```bash
ssh azureuser@74.248.37.10
```

---

## 4.5 `ansible_ssh_private_key_file`

```ini
ansible_ssh_private_key_file=/home/bel-idri/Desktop/id_rsa
```

This tells Ansible which private SSH key to use for authentication.

Conceptually this is similar to:

```bash
ssh -i /home/bel-idri/Desktop/id_rsa azureuser@74.248.37.10
```

The private key is used for SSH authentication. It is not the same thing as the Ansible Vault password.

Two different secrets are involved:

```text
SSH private key
    ↓
Authenticates to the Ubuntu server

Vault password
    ↓
Decrypts vault.yml locally for Ansible
```

---

## 4.6 `domain_name`

```ini
domain_name=abel-hid.duckdns.org
```

This is a custom variable used in several places.

It is used by:

```text
TLS / Certbot
Nginx server_name
WordPress installation URL
phpMyAdmin absolute URL
```

For example:

```jinja
server_name {{ domain_name }};
```

becomes:

```nginx
server_name abel-hid.duckdns.org;
```

And:

```jinja
-d {{ domain_name }}
```

becomes:

```text
-d abel-hid.duckdns.org
```

for Certbot.

---

## 4.7 `use_letsencrypt=True`

```ini
use_letsencrypt=True
```

This variable decides which web mode is generated.

### When it is `True`

```text
TLS role runs
      ↓
Certbot certificate exists or is created
      ↓
Compose publishes 80 and 443
      ↓
Nginx redirects HTTP to HTTPS
      ↓
Nginx serves HTTPS using the certificate
```

### When it is `False`

```text
TLS tasks are skipped
      ↓
Compose publishes only port 80
      ↓
Nginx listens on port 80
      ↓
Site works through HTTP
```

Again: the current code does **not** generate a self-signed certificate when this is false.

---

# 5. Variable precedence: why `vars.yaml` says false but inventory says true

The final `vars.yaml` contains:

```yaml
use_letsencrypt: false
```

but the host in `inventory.ini` contains:

```ini
use_letsencrypt=True
```

For this host, the inventory host variable overrides the group variable.

So for your current host the effective value is:

```text
use_letsencrypt = True
```

Think of the `false` in `vars.yaml` as the default for hosts that do not override it.

Current host:

```text
vars.yaml
use_letsencrypt: false
        ↓
        overridden by
        ↓
inventory.ini
use_letsencrypt=True
        ↓
FINAL VALUE = True
```

If you change the inventory to:

```ini
use_letsencrypt=False
```

then the final value becomes false for that host.

---

# 6. `group_vars/all/vars.yaml`

The final file is:

```yaml
app_dir: /opt/cloud1

use_letsencrypt: false

site_scheme: "{{ 'https' if use_letsencrypt else 'http' }}"

letsencrypt_email: abdeljalilelhidaoui21@gmail.com

mysql_database: wordpress
mysql_user: wp_user
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_password: "{{ vault_mysql_password }}"

wordpress_title: Cloud-1
wordpress_admin_user: cloud1_admin
wordpress_admin_email: abdeljalilelhidaoui21@gmail.com
wordpress_admin_password: "{{ vault_wordpress_admin_password }}"
```

Because the directory is:

```text
group_vars/all/
```

these variables are automatically associated with all hosts.

---

## 6.1 `app_dir`

```yaml
app_dir: /opt/cloud1
```

This is the deployment directory on the remote server.

Ansible renders files there:

```text
/opt/cloud1/
├── docker-compose.yml
├── nginx.conf
└── certs/          # when Let's Encrypt is enabled
    ├── fullchain.pem
    └── privkey.pem
```

Instead of repeating `/opt/cloud1` everywhere, the code uses:

```jinja
{{ app_dir }}
```

For example:

```yaml
path: "{{ app_dir }}"
```

becomes:

```yaml
path: /opt/cloud1
```

---

## 6.2 `use_letsencrypt`

```yaml
use_letsencrypt: false
```

This is the default TLS switch.

The inventory currently overrides it to `True` for `cloud1-azure`.

The value is consumed by:

```text
TLS role
Docker Compose template
Nginx template
site_scheme
```

---

## 6.3 `site_scheme`

```yaml
site_scheme: "{{ 'https' if use_letsencrypt else 'http' }}"
```

This is a Jinja conditional expression.

Read it as:

```text
if use_letsencrypt is true
    site_scheme = https
else
    site_scheme = http
```

So:

```text
True  → https
False → http
```

This value is later used for:

```text
WordPress URL
phpMyAdmin absolute URL
```

For example:

```jinja
--url={{ site_scheme }}://{{ domain_name }}
```

with TLS enabled becomes:

```text
--url=https://abel-hid.duckdns.org
```

With TLS disabled and `domain_name` set to an IP, it could become:

```text
--url=http://74.248.37.10
```

---

## 6.4 `letsencrypt_email`

```yaml
letsencrypt_email: abdeljalilelhidaoui21@gmail.com
```

Certbot uses this value here:

```jinja
-m {{ letsencrypt_email }}
```

The `-m` option supplies the account/contact email to Certbot.

This is configuration, not the TLS private key.

---

## 6.5 MySQL variables

```yaml
mysql_database: wordpress
mysql_user: wp_user
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_password: "{{ vault_mysql_password }}"
```

### Database name

```yaml
mysql_database: wordpress
```

This means the MySQL container creates a database named:

```text
wordpress
```

### WordPress database user

```yaml
mysql_user: wp_user
```

This is the normal database account WordPress uses.

WordPress does not need to connect as MySQL root.

### Root password

```yaml
mysql_root_password: "{{ vault_mysql_root_password }}"
```

This variable does not contain the password itself.

It references the encrypted Vault variable:

```text
vault_mysql_root_password
```

### WordPress database password

```yaml
mysql_password: "{{ vault_mysql_password }}"
```

Same idea:

```text
normal variable
mysql_password
      ↓
gets value from encrypted variable
vault_mysql_password
```

---

## 6.6 WordPress variables

```yaml
wordpress_title: Cloud-1
wordpress_admin_user: cloud1_admin
wordpress_admin_email: abdeljalilelhidaoui21@gmail.com
wordpress_admin_password: "{{ vault_wordpress_admin_password }}"
```

These values are used during the automated initial WordPress installation.

Instead of opening WordPress in the browser and completing the installation form manually, WP-CLI receives these values.

---

# 7. `vault.yml`

The final file begins with something like:

```text
$ANSIBLE_VAULT;1.1;AES256
...
```

That means the file is encrypted by Ansible Vault.

The encrypted file contains secret variables such as conceptually:

```yaml
vault_mysql_root_password: "..."
vault_mysql_password: "..."
vault_wordpress_admin_password: "..."
```

You cannot read them normally with:

```bash
cat group_vars/all/vault.yml
```

because you only see encrypted text.

---

# 8. Vault password versus application passwords

There are two levels.

```text
PASS.txt
    │
    │ Vault password
    │ decrypts
    ▼
vault.yml
    │
    ├── MySQL root password
    ├── MySQL WordPress-user password
    └── WordPress administrator password
```

`PASS.txt` is **not** the MySQL password.

It is the password that unlocks the encrypted Vault file.

You can view the encrypted variables with:

```bash
ansible-vault view group_vars/all/vault.yml \
  --vault-password-file ~/Desktop/PASS.txt
```

And your deployment command can provide the same Vault password file:

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Desktop/PASS.txt
```

---

# 9. Jinja syntax: `{{ ... }}`

You see syntax like:

```jinja
{{ app_dir }}
```

```jinja
{{ mysql_database }}
```

```jinja
{{ domain_name }}
```

This means:

> Evaluate this variable/expression and put its value here.

Example:

```yaml
app_dir: /opt/cloud1
```

and:

```yaml
path: "{{ app_dir }}"
```

becomes effectively:

```yaml
path: /opt/cloud1
```

Jinja is used in both Ansible YAML and the `.j2` templates.

---

# 10. `site.yml` — the main playbook

The final file is:

```yaml
- name: Deploy Cloud-1 WordPress stack
  hosts: webserver
  become: true
  roles:
    - docker
    - firewall
    - tls
    - wordpress
```

This is the deployment entry point.

When you run:

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Desktop/PASS.txt
```

Ansible starts from this file.

---

## 10.1 `name`

```yaml
name: Deploy Cloud-1 WordPress stack
```

This is a human-readable description.

It produces output such as:

```text
PLAY [Deploy Cloud-1 WordPress stack]
```

---

## 10.2 `hosts: webserver`

```yaml
hosts: webserver
```

This means:

> Run against every host in the `[webserver]` inventory group.

---

## 10.3 `become: true`

```yaml
become: true
```

Ansible connects as:

```text
azureuser
```

but many deployment operations need root privileges:

```text
apt
/etc/apt
systemd
UFW
/opt/cloud1
Docker administration
/etc/letsencrypt
```

`become: true` means Ansible performs privilege escalation on the **remote server**, normally through sudo.

Conceptually:

```text
SSH as azureuser
      ↓
become: true
      ↓
root privileges for deployment tasks
```

---

## 10.4 Role order

The roles run in this order:

```text
1. docker
2. firewall
3. tls
4. wordpress
```

Why?

```text
Docker
  ↓
Docker/Compose must exist before application commands run

Firewall
  ↓
External network access is restricted

TLS
  ↓
Certificate is prepared before HTTPS Nginx starts

WordPress
  ↓
Compose + Nginx are rendered and containers start
```

---

# 11. What is an Ansible role?

A role is a logical group of related tasks.

Instead of putting everything in one huge playbook:

```text
site.yml with hundreds of lines
```

this project has:

```text
docker role
firewall role
tls role
wordpress role
```

`site.yml` acts like the coordinator.

```text
site.yml
   |
   +-- docker
   |
   +-- firewall
   |
   +-- tls
   |
   +-- wordpress
```

---

# 12. Docker role

File:

```text
roles/docker/tasks/main.yml
```

Its job is to install Docker Engine and the Docker Compose plugin and make sure Docker starts automatically.

The final order is:

```text
Install repository prerequisites
        ↓
Create keyring directory
        ↓
Download Docker signing key
        ↓
Detect machine architecture
        ↓
Remove old repository file
        ↓
Configure Docker repository
        ↓
Install Docker packages
        ↓
Enable/start Docker service
```

---

## 12.1 Install repository prerequisites

```yaml
- name: Install packages needed for Docker's apt repository
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
    state: present
    update_cache: true
```

`ansible.builtin.apt` manages packages on Ubuntu/Debian.

```yaml
state: present
```

means:

> Ensure these packages are installed.

It does not mean blindly install them again every run.

```yaml
update_cache: true
```

means Ansible updates the APT package index before installing them.

The packages are:

```text
ca-certificates
    ↓
Allows the system to trust HTTPS certificate authorities

curl
    ↓
General HTTP/HTTPS transfer utility needed as part of repository setup
```

---

## 12.2 Create `/etc/apt/keyrings`

```yaml
- name: Create apt keyrings directory
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: "0755"
```

`ansible.builtin.file` manages filesystem objects.

```yaml
state: directory
```

means:

> Ensure this path exists as a directory.

```yaml
mode: "0755"
```

is its Unix permission mode.

---

## 12.3 Download Docker's signing key

```yaml
- name: Download Docker's official signing key
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: "0644"
```

`ansible.builtin.get_url` downloads a remote file.

The key is used so APT can verify packages from Docker's repository.

---

## 12.4 Read the package architecture

```yaml
- name: Read Debian package architecture
  ansible.builtin.command: dpkg --print-architecture
  register: docker_dpkg_arch
  changed_when: false
```

The command can return something such as:

```text
amd64
```

The result is saved using:

```yaml
register: docker_dpkg_arch
```

That creates a variable containing information about the command result.

Later the code uses:

```jinja
{{ docker_dpkg_arch.stdout }}
```

`stdout` is the command's standard output.

### Why `changed_when: false`?

The command only reads information.

It does not change the server.

So:

```yaml
changed_when: false
```

forces Ansible to report this task as a check rather than a change.

---

## 12.5 Remove an old Docker repository file

```yaml
- name: Remove old Docker repository file if present
  ansible.builtin.file:
    path: /etc/apt/sources.list.d/docker.list
    state: absent
```

```yaml
state: absent
```

means:

> Ensure this path does not exist.

The current code uses the newer:

```text
/etc/apt/sources.list.d/docker.sources
```

so it removes the older `docker.list` file if one is present.

---

## 12.6 Configure the Docker APT repository

```yaml
- name: Configure Docker apt repository
  ansible.builtin.copy:
    dest: /etc/apt/sources.list.d/docker.sources
    mode: "0644"
    content: |
      Types: deb
      URIs: https://download.docker.com/linux/ubuntu
      Suites: {{ ansible_distribution_release }}
      Components: stable
      Architectures: {{ docker_dpkg_arch.stdout }}
      Signed-By: /etc/apt/keyrings/docker.asc
```

Here `ansible.builtin.copy` is used with inline `content`.

So Ansible creates the repository configuration file directly.

### `content: |`

The `|` means the following indented lines are one multi-line string.

### `ansible_distribution_release`

This is an Ansible fact gathered from the remote machine.

For Ubuntu 22.04 it is normally related to the release codename.

### `docker_dpkg_arch.stdout`

This comes from the earlier architecture command.

So the repository is adapted to the current machine instead of hard-coding one architecture.

---

## 12.7 Install Docker

```yaml
- name: Install Docker Engine and Docker Compose plugin
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: true
```

The important parts are:

```text
docker-ce
    ↓
Docker Engine

docker-ce-cli
    ↓
Docker command-line client

containerd.io
    ↓
Container runtime component used by Docker

docker-buildx-plugin
    ↓
Docker Buildx plugin installed by the current role

docker-compose-plugin
    ↓
Provides the `docker compose` command
```

The project later uses commands such as:

```bash
docker compose up -d
```

so the Compose plugin is required by this code.

---

## 12.8 Enable and start Docker

```yaml
- name: Ensure Docker service is enabled and running
  ansible.builtin.systemd:
    name: docker
    state: started
    enabled: true
```

Two things happen:

```yaml
state: started
```

means:

> Docker should be running now.

```yaml
enabled: true
```

means:

> Docker should start automatically during server boot.

This later works together with Compose service restart policies.

---

# 13. Firewall role

File:

```text
roles/firewall/tasks/main.yml
```

The role uses UFW.

Its final desired public policy is:

```text
Incoming default: DENY

Allowed TCP ports:
22  SSH
80  HTTP
443 HTTPS
```

---

## 13.1 Install UFW

```yaml
- name: Install UFW
  ansible.builtin.apt:
    name: ufw
    state: present
```

This ensures the firewall utility exists.

---

## 13.2 Deny incoming connections by default

```yaml
- name: Deny incoming connections by default
  ansible.builtin.command: ufw default deny incoming
  changed_when: false
```

This defines the default incoming policy.

Conceptually:

```text
Someone tries to connect to a port
        ↓
Is there an allow rule?
   │          │
  yes         no
   │          │
 allow       deny
```

The current task uses:

```yaml
changed_when: false
```

so Ansible does not count this command as a changed task in its recap.

---

## 13.3 Allow outgoing connections

```yaml
- name: Allow outgoing connections by default
  ansible.builtin.command: ufw default allow outgoing
  changed_when: false
```

This means processes on the server are allowed to initiate outgoing connections by default.

For example the server needs outgoing access to download packages and container images.

---

## 13.4 Allow ports 22, 80 and 443

```yaml
- name: Allow only SSH, HTTP and HTTPS from outside
  ansible.builtin.command: "ufw allow {{ item }}/tcp"
  loop:
    - "22"
    - "80"
    - "443"
  register: ufw_rule
  changed_when: "'Skipping adding existing rule' not in ufw_rule.stdout"
```

### `loop`

Instead of writing three separate tasks:

```text
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
```

Ansible repeats one task three times.

Each time:

```jinja
{{ item }}
```

has the current value.

Iteration 1:

```text
item = 22
```

Iteration 2:

```text
item = 80
```

Iteration 3:

```text
item = 443
```

### `register: ufw_rule`

The command result is stored so its output can be inspected.

### Custom `changed_when`

```yaml
changed_when: "'Skipping adding existing rule' not in ufw_rule.stdout"
```

If UFW says the rule already exists, the task is treated as unchanged.

If it actually adds a rule, the task is treated as changed.

This helps make repeated runs more meaningful.

---

## 13.5 Enable UFW

```yaml
- name: Enable UFW
  ansible.builtin.command: ufw --force enable
  changed_when: false
```

`--force` avoids an interactive confirmation prompt.

That matters because an automated playbook cannot wait for a human to type `y`.

---

# 14. TLS role

File:

```text
roles/tls/tasks/main.yml
```

This role is only active when:

```yaml
use_letsencrypt
```

is true.

Every task in the current file has either:

```yaml
when: use_letsencrypt
```

or a condition that includes it.

So:

```text
use_letsencrypt=False
        ↓
TLS role tasks are skipped
        ↓
No self-signed certificate is created
```

---

## 14.1 Create application directory

```yaml
- name: Create application directory
  ansible.builtin.file:
    path: "{{ app_dir }}"
    state: directory
    mode: "0755"
  when: use_letsencrypt
```

With the current variable:

```yaml
app_dir: /opt/cloud1
```

this ensures:

```text
/opt/cloud1
```

exists before certificate files are copied there.

---

## 14.2 Create certificate directory

```yaml
- name: Create certificate directory
  ansible.builtin.file:
    path: "{{ app_dir }}/certs"
    state: directory
    mode: "0755"
  when: use_letsencrypt
```

This creates:

```text
/opt/cloud1/certs
```

Nginx later mounts this directory read-only inside its container.

---

## 14.3 Install Certbot

```yaml
- name: Install Certbot
  ansible.builtin.apt:
    name: certbot
    state: present
  when: use_letsencrypt
```

Certbot is the program used to request/manage the Let's Encrypt certificate.

---

## 14.4 Check whether the certificate already exists

```yaml
- name: Check whether Let's Encrypt certificate already exists
  ansible.builtin.stat:
    path: "/etc/letsencrypt/live/{{ domain_name }}/fullchain.pem"
  register: letsencrypt_cert
  when: use_letsencrypt
```

`ansible.builtin.stat` checks information about a path.

It does not create the file.

The result is stored in:

```text
letsencrypt_cert
```

Later the code checks:

```jinja
letsencrypt_cert.stat.exists
```

So the logic is:

```text
Does certificate exist?
      │
  yes │ no
      │
      ├── yes → do not request another one
      │
      └── no  → run Certbot
```

This is part of making repeated deployment safe.

---

## 14.5 Check whether an old Compose file exists

```yaml
- name: Check whether an old Compose file already exists
  ansible.builtin.stat:
    path: "{{ app_dir }}/docker-compose.yml"
  register: existing_compose
  when: use_letsencrypt
```

This supports the case where the same VM already has the stack running.

For example, you may first deploy:

```text
use_letsencrypt=False
```

and later switch to:

```text
use_letsencrypt=True
```

If old Nginx is already using port 80, Certbot standalone cannot use that port at the same time.

So the role checks whether there is already a Compose deployment before trying to stop Nginx.

---

## 14.6 Stop existing Nginx if Certbot needs port 80

```yaml
- name: Stop existing Nginx before Certbot uses port 80
  ansible.builtin.command: docker compose stop nginx
  args:
    chdir: "{{ app_dir }}"
  when:
    - use_letsencrypt
    - not letsencrypt_cert.stat.exists
    - existing_compose.stat.exists
```

All three conditions must be true.

```text
use_letsencrypt = true
AND
certificate does not exist
AND
old Compose file exists
```

Only then does Ansible run:

```bash
docker compose stop nginx
```

### `chdir`

```yaml
chdir: "{{ app_dir }}"
```

means execute the command from:

```text
/opt/cloud1
```

That is where the generated `docker-compose.yml` lives.

---

## 14.7 Obtain the Let's Encrypt certificate

```yaml
- name: Obtain Let's Encrypt certificate
  ansible.builtin.command: >-
    certbot certonly --standalone
    -d {{ domain_name }}
    --non-interactive
    --agree-tos
    -m {{ letsencrypt_email }}
  when:
    - use_letsencrypt
    - not letsencrypt_cert.stat.exists
```

### `>-`

This YAML syntax lets one shell command be written across several YAML lines.

The resulting command is conceptually:

```bash
certbot certonly --standalone \
  -d abel-hid.duckdns.org \
  --non-interactive \
  --agree-tos \
  -m abdeljalilelhidaoui21@gmail.com
```

### `certonly`

Request/manage the certificate without asking Certbot to configure Nginx automatically.

Your own Nginx template is responsible for Nginx configuration.

### `--standalone`

Certbot temporarily serves its own challenge server to prove control of the domain.

### `--non-interactive`

Do not ask terminal questions.

This is required for automation.

### `--agree-tos`

Accept the service terms non-interactively.

### `-d`

The domain whose certificate is requested.

### `-m`

Contact/account email.

---

## 14.8 Copy the public certificate chain

```yaml
- name: Copy Let's Encrypt certificate
  ansible.builtin.copy:
    src: "/etc/letsencrypt/live/{{ domain_name }}/fullchain.pem"
    dest: "{{ app_dir }}/certs/fullchain.pem"
    remote_src: true
    mode: "0644"
  when: use_letsencrypt
```

Normally `copy` copies from the Ansible control machine.

But:

```yaml
remote_src: true
```

means the source already exists on the remote server.

So this operation is:

```text
REMOTE SERVER
/etc/letsencrypt/live/.../fullchain.pem
             ↓ copy
/opt/cloud1/certs/fullchain.pem
```

---

## 14.9 Copy the certificate private key

```yaml
- name: Copy Let's Encrypt private key
  ansible.builtin.copy:
    src: "/etc/letsencrypt/live/{{ domain_name }}/privkey.pem"
    dest: "{{ app_dir }}/certs/privkey.pem"
    remote_src: true
    mode: "0600"
  when: use_letsencrypt
```

The destination mode is:

```text
0600
```

which means the file is restricted to its owner for reading/writing.

Nginx receives it through a read-only Docker bind mount later.

---

# 15. What happens in TLS false mode?

If the inventory contains:

```ini
use_letsencrypt=False
```

then all TLS tasks are skipped.

No Certbot installation is required by this role.

No certificate is generated.

No `/opt/cloud1/certs` certificate is needed.

Then the templates generate HTTP-only behavior:

```text
Nginx host port 80
        ↓
Nginx listens on 80
        ↓
WordPress URL uses http://
```

This is why your HTTP test returned:

```text
HTTP/1.1 200 OK
```

while port 443 could be closed/refused in false mode.

---

# 16. WordPress role

File:

```text
roles/wordpress/tasks/main.yml
```

Despite the role name, this role orchestrates the complete application stack from the Compose template:

```text
MySQL
WordPress PHP-FPM
phpMyAdmin
Nginx
WP-CLI helper
```

The task sequence is:

```text
Create /opt/cloud1
      ↓
Render docker-compose.yml
      ↓
Render nginx.conf
      ↓
Start long-running containers
      ↓
Wait for MySQL
      ↓
Wait for WordPress files
      ↓
Check if WordPress is installed
      ↓
Install WordPress only if needed
```

---

## 16.1 Create application directory

```yaml
- name: Create application directory
  ansible.builtin.file:
    path: "{{ app_dir }}"
    state: directory
    mode: "0755"
```

This ensures `/opt/cloud1` exists even in HTTP mode where the TLS role was skipped.

That is important because:

```text
use_letsencrypt=False
        ↓
TLS role does not create /opt/cloud1
        ↓
WordPress role still creates it
```

---

## 16.2 Render `docker-compose.yml`

```yaml
- name: Render docker-compose.yml
  ansible.builtin.template:
    src: docker-compose.yml.j2
    dest: "{{ app_dir }}/docker-compose.yml"
    mode: "0600"
  diff: false
```

`ansible.builtin.template` takes a local Jinja template and renders variables into it.

Source on your local machine:

```text
roles/wordpress/templates/docker-compose.yml.j2
```

Destination on Azure:

```text
/opt/cloud1/docker-compose.yml
```

Flow:

```text
LOCAL TEMPLATE
      ↓
Jinja variables evaluated by Ansible
      ↓
REMOTE GENERATED FILE
```

### Why mode `0600`?

The rendered Compose file contains actual database password values in environment variables.

`0600` restricts access to the owner.

### Why `diff: false`?

Ansible can normally show template diffs when content changes.

Because this rendered file contains secrets, disabling the diff avoids exposing password changes in diff output.

---

## 16.3 Render Nginx configuration

```yaml
- name: Render Nginx configuration
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: "{{ app_dir }}/nginx.conf"
    mode: "0644"
  notify: Restart nginx container
```

Source:

```text
roles/wordpress/templates/nginx.conf.j2
```

Destination:

```text
/opt/cloud1/nginx.conf
```

### `notify`

```yaml
notify: Restart nginx container
```

means:

> If this template task changes the file, notify the handler with this name.

The handler is explained later.

---

## 16.4 Start all long-running containers

```yaml
- name: Start all long-running containers
  ansible.builtin.command: docker compose up -d
  args:
    chdir: "{{ app_dir }}"
  register: compose_up
  changed_when: >-
    'Created' in compose_up.stderr or
    'Recreated' in compose_up.stderr or
    'Started' in compose_up.stderr
```

The command is:

```bash
docker compose up -d
```

### `up`

Create/start the services described by `docker-compose.yml`.

### `-d`

Detached mode. The command returns instead of attaching your terminal to container logs.

### Why doesn't `wpcli` start here?

`wpcli` has a Compose profile:

```yaml
profiles:
  - tools
```

So normal:

```bash
docker compose up -d
```

starts the normal services, not the optional tools-profile container.

### `register: compose_up`

Stores the Docker Compose command result.

### Custom `changed_when`

The task is reported as changed only if Compose output indicates containers were actually:

```text
Created
Recreated
Started
```

If everything is already running and nothing changes, the task can report unchanged.

---

# 17. Why wait after `docker compose up -d`?

A container being started does not mean the application inside it is immediately ready.

Example MySQL timeline:

```text
Docker starts MySQL container
        ↓
MySQL process starts
        ↓
Database files initialize
        ↓
Users/database initialize
        ↓
MySQL finally accepts connections
```

So the playbook waits for readiness instead of assuming immediate availability.

---

## 17.1 Wait for MySQL

```yaml
- name: Wait until MySQL accepts connections
  ansible.builtin.command:
    argv:
      - docker
      - compose
      - exec
      - -T
      - mysql
      - sh
      - -c
      - 'mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD" --silent'
  args:
    chdir: "{{ app_dir }}"
  register: mysql_ready
  changed_when: false
  retries: 30
  delay: 2
  until: mysql_ready.rc == 0
```

This is easier to understand as the equivalent command:

```bash
docker compose exec -T mysql sh -c \
  'mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD" --silent'
```

### `argv`

Instead of writing one shell command string, Ansible receives each argument separately.

This avoids some shell parsing problems on the outer command.

### `docker compose exec`

Run a command inside an already-running service container.

### `-T`

Disable allocation of a pseudo-TTY.

That is useful for non-interactive automation.

### `mysql`

This is the Compose service name.

### `sh -c`

Run the following string through a shell inside the MySQL container.

That is important because the command uses:

```text
$MYSQL_ROOT_PASSWORD
```

which is an environment variable inside the container.

### `mysqladmin ping`

Checks whether the MySQL server is accepting connections.

### `register: mysql_ready`

Stores the command result.

### `rc`

```text
rc = return code
```

Unix convention:

```text
0     = success
non-0 = failure / not ready
```

### Retry logic

```yaml
retries: 30
delay: 2
until: mysql_ready.rc == 0
```

means:

```text
try
 ↓
not ready?
 ↓
wait 2 seconds
 ↓
try again
 ↓
up to 30 attempts
 ↓
continue when rc == 0
```

### `changed_when: false`

This is only a readiness check. It does not change the desired system configuration.

---

## 17.2 Wait for WordPress files

```yaml
- name: Wait until WordPress files are initialized
  ansible.builtin.command:
    argv:
      - docker
      - compose
      - exec
      - -T
      - wordpress
      - test
      - -f
      - /var/www/html/wp-config.php
  args:
    chdir: "{{ app_dir }}"
  register: wordpress_files
  changed_when: false
  retries: 30
  delay: 2
  until: wordpress_files.rc == 0
```

Equivalent idea:

```bash
docker compose exec -T wordpress \
  test -f /var/www/html/wp-config.php
```

`test -f` checks whether the file exists as a regular file.

Why this matters:

```text
wordpress container starts
        ↓
WordPress image initializes files in shared volume
        ↓
wp-config.php appears
        ↓
WP-CLI can safely use the WordPress installation files
```

---

# 18. Check whether WordPress is already installed

```yaml
- name: Check whether WordPress is already installed
  ansible.builtin.command: >-
    docker compose --profile tools run --rm wpcli
    wp core is-installed --path=/var/www/html
  args:
    chdir: "{{ app_dir }}"
  register: wordpress_installed
  changed_when: false
  failed_when: false
```

The important command is:

```bash
docker compose --profile tools run --rm wpcli \
  wp core is-installed --path=/var/www/html
```

---

## 18.1 `--profile tools`

The `wpcli` service belongs to:

```yaml
profiles:
  - tools
```

So this enables that optional service for this command.

---

## 18.2 `run`

```bash
docker compose run ... wpcli
```

creates a one-off container from the `wpcli` service definition.

WP-CLI is not intended to stay running as a permanent server.

---

## 18.3 `--rm`

Delete the temporary WP-CLI container when the command finishes.

This does **not** delete the shared WordPress volume.

---

## 18.4 `wp core is-installed`

This asks WordPress:

> Is the site already installed in the database?

If installed:

```text
rc = 0
```

If not installed:

```text
rc != 0
```

The result is stored as:

```text
wordpress_installed
```

---

## 18.5 Why `failed_when: false`?

A non-zero result here can simply mean:

```text
WordPress is not installed yet
```

That is not an unexpected error on the first deployment.

It is information used by the next task.

Therefore:

```yaml
failed_when: false
```

means:

> Do not abort the whole play just because this check says WordPress is not installed.

---

# 19. Install the initial WordPress site

```yaml
- name: Install the initial WordPress site
  ansible.builtin.command: >-
    docker compose --profile tools run --rm wpcli
    wp core install
    --path=/var/www/html
    --url={{ site_scheme }}://{{ domain_name }}
    --title={{ wordpress_title | quote }}
    --admin_user={{ wordpress_admin_user | quote }}
    --admin_password={{ wordpress_admin_password | quote }}
    --admin_email={{ wordpress_admin_email | quote }}
    --skip-email
  args:
    chdir: "{{ app_dir }}"
  when: wordpress_installed.rc != 0
  no_log: true
```

This replaces the normal browser-based WordPress installation form.

Instead of manually entering:

```text
site URL
title
admin username
admin password
admin email
```

WP-CLI performs the installation automatically.

---

## 19.1 `when: wordpress_installed.rc != 0`

This is crucial for idempotency.

First deployment:

```text
WordPress not installed
        ↓
rc != 0
        ↓
run wp core install
```

Second deployment:

```text
WordPress already installed
        ↓
rc == 0
        ↓
SKIP wp core install
```

Without this condition, the playbook would try to install WordPress again on every run.

---

## 19.2 URL

```jinja
--url={{ site_scheme }}://{{ domain_name }}
```

TLS true:

```text
https://abel-hid.duckdns.org
```

TLS false with an IP domain value:

```text
http://74.248.37.10
```

---

## 19.3 `| quote`

Example:

```jinja
{{ wordpress_title | quote }}
```

`quote` safely quotes the value for use in a command argument.

This is useful if a value contains spaces or shell-sensitive characters.

---

## 19.4 `--skip-email`

WordPress installation does not send the installation email.

---

## 19.5 `no_log: true`

This task contains:

```text
WordPress admin password
```

So:

```yaml
no_log: true
```

prevents Ansible from printing sensitive task arguments/results into logs.

This is why a failure in this task may show censored output instead of the full command.

---

# 20. WordPress handler

File:

```text
roles/wordpress/handlers/main.yml
```

Final content:

```yaml
- name: Restart nginx container
  ansible.builtin.command: docker compose restart nginx
  args:
    chdir: "{{ app_dir }}"
```

A handler is a task that normally runs when another task notifies it.

The Nginx template task contains:

```yaml
notify: Restart nginx container
```

So:

```text
Did nginx.conf change?
       │
       ├── no  → no handler needed
       │
       └── yes → notify handler
                  ↓
             restart nginx
```

This lets Nginx load the new configuration.

---

# 21. Docker Compose template

File:

```text
roles/wordpress/templates/docker-compose.yml.j2
```

This is not the final Compose file directly.

It is a **Jinja template**.

Ansible renders it to:

```text
/opt/cloud1/docker-compose.yml
```

The final services are:

```text
mysql
wordpress
phpmyadmin
nginx
wpcli
```

`wpcli` is a temporary tool service, so normally only four long-running application containers remain.

---

# 22. MySQL service

Template:

```yaml
mysql:
  image: mysql:8.0
  container_name: cloud1_mysql
  restart: unless-stopped
  environment:
    MYSQL_ROOT_PASSWORD: "{{ mysql_root_password }}"
    MYSQL_DATABASE: "{{ mysql_database }}"
    MYSQL_USER: "{{ mysql_user }}"
    MYSQL_PASSWORD: "{{ mysql_password }}"
  volumes:
    - mysql_data:/var/lib/mysql
  networks:
    - backend
```

---

## 22.1 Image

```yaml
image: mysql:8.0
```

Docker uses the official MySQL 8.0 image.

---

## 22.2 `container_name`

```yaml
container_name: cloud1_mysql
```

This explicitly names the created container:

```text
cloud1_mysql
```

The Compose service name remains:

```text
mysql
```

Other Compose services use the service name for Docker DNS.

---

## 22.3 `restart: unless-stopped`

```yaml
restart: unless-stopped
```

Broadly means:

> Automatically restart this service after Docker/server restart unless an administrator deliberately stopped it.

This works together with the Docker systemd service being enabled at boot.

---

## 22.4 Environment variables

The MySQL image reads these variables during initialization:

```text
MYSQL_ROOT_PASSWORD
MYSQL_DATABASE
MYSQL_USER
MYSQL_PASSWORD
```

They create/configure the initial database and users.

---

## 22.5 MySQL volume

```yaml
- mysql_data:/var/lib/mysql
```

Inside the container, MySQL data is stored at:

```text
/var/lib/mysql
```

Instead of keeping that only in the disposable container filesystem, it is connected to the named volume:

```text
mysql_data
```

So:

```text
MySQL container
     ↓
mysql_data volume
     ↓
data survives container recreation
```

---

## 22.6 MySQL network

```yaml
networks:
  - backend
```

MySQL is only attached to the backend Docker network.

There is no host `ports:` section for MySQL.

So the Compose file does not publish MySQL's port directly to the Internet.

---

# 23. WordPress PHP-FPM service

```yaml
wordpress:
  image: wordpress:php8.2-fpm
  container_name: cloud1_wordpress
  restart: unless-stopped
  depends_on:
    - mysql
  environment:
    WORDPRESS_DB_HOST: mysql
    WORDPRESS_DB_NAME: "{{ mysql_database }}"
    WORDPRESS_DB_USER: "{{ mysql_user }}"
    WORDPRESS_DB_PASSWORD: "{{ mysql_password }}"
  volumes:
    - wordpress_data:/var/www/html
  networks:
    - backend
    - frontend
```

---

## 23.1 Why `wordpress:php8.2-fpm`?

This container runs PHP through PHP-FPM.

It is not the public web server.

Nginx is the public web server and forwards `.php` requests to WordPress PHP-FPM.

Conceptually:

```text
Browser
   ↓
Nginx
   ↓ FastCGI
wordpress:9000
```

---

## 23.2 `depends_on`

```yaml
depends_on:
  - mysql
```

This expresses a startup dependency at the Compose level.

But it does not prove MySQL is fully ready to accept queries.

That is why the Ansible role still has the explicit MySQL readiness loop.

---

## 23.3 `WORDPRESS_DB_HOST: mysql`

Why is the host simply:

```text
mysql
```

instead of an IP?

Because Docker Compose provides internal DNS using service names.

The service is declared as:

```yaml
services:
  mysql:
```

So other services on the same Docker network can resolve:

```text
mysql
```

to the current MySQL container address.

This is better than hard-coding a container IP because container IPs can change.

---

## 23.4 WordPress volume

```yaml
- wordpress_data:/var/www/html
```

WordPress files live in a named volume so they persist independently of the individual container.

The same volume is shared with:

```text
nginx    → read-only
wpcli    → read/write
```

---

## 23.5 WordPress networks

```yaml
networks:
  - backend
  - frontend
```

WordPress needs:

```text
backend
    ↓
to communicate with MySQL

frontend
    ↓
to communicate with Nginx
```

---

# 24. phpMyAdmin service

```yaml
phpmyadmin:
  image: phpmyadmin:5.2
  container_name: cloud1_phpmyadmin
  restart: unless-stopped
  depends_on:
    - mysql
  environment:
    PMA_HOST: mysql
    PMA_ABSOLUTE_URI: "{{ site_scheme }}://{{ domain_name }}/phpmyadmin/"
  networks:
    - backend
    - frontend
```

---

## 24.1 `PMA_HOST`

```yaml
PMA_HOST: mysql
```

phpMyAdmin connects to the MySQL Compose service by Docker DNS name.

---

## 24.2 `PMA_ABSOLUTE_URI`

```jinja
{{ site_scheme }}://{{ domain_name }}/phpmyadmin/
```

TLS true:

```text
https://abel-hid.duckdns.org/phpmyadmin/
```

TLS false:

```text
http://.../phpmyadmin/
```

This tells phpMyAdmin its externally visible URL behind Nginx.

---

## 24.3 Networks

phpMyAdmin needs both:

```text
frontend
    ↓
Nginx can proxy requests to it

backend
    ↓
It can reach MySQL
```

---

# 25. Nginx service

```yaml
nginx:
  image: nginx:alpine
  container_name: cloud1_nginx
  restart: unless-stopped
  depends_on:
    - wordpress
    - phpmyadmin
```

Nginx is the public entry point.

---

## 25.1 Conditional ports

The template contains:

```yaml
ports:
  - "80:80"
{% if use_letsencrypt %}
  - "443:443"
{% endif %}
```

This is important.

### TLS true

Rendered Compose:

```yaml
ports:
  - "80:80"
  - "443:443"
```

### TLS false

Rendered Compose:

```yaml
ports:
  - "80:80"
```

So false mode does not publish host port 443 through this Nginx service.

---

## 25.2 What does `"80:80"` mean?

```text
HOST PORT : CONTAINER PORT
    80    :      80
```

So traffic arriving at the VM's port 80 is forwarded to Nginx container port 80.

Same for 443 when enabled.

---

## 25.3 Nginx volumes

```yaml
volumes:
  - wordpress_data:/var/www/html:ro
  - {{ app_dir }}/nginx.conf:/etc/nginx/conf.d/default.conf:ro
{% if use_letsencrypt %}
  - {{ app_dir }}/certs:/etc/nginx/certs:ro
{% endif %}
```

### WordPress files

```yaml
wordpress_data:/var/www/html:ro
```

Nginx can read the WordPress files.

`:ro` means read-only.

### Nginx configuration

```yaml
{{ app_dir }}/nginx.conf:/etc/nginx/conf.d/default.conf:ro
```

Remote host file:

```text
/opt/cloud1/nginx.conf
```

is mounted into the Nginx container as:

```text
/etc/nginx/conf.d/default.conf
```

read-only.

### Certificates

When TLS is true:

```yaml
{{ app_dir }}/certs:/etc/nginx/certs:ro
```

maps:

```text
REMOTE HOST
/opt/cloud1/certs/
       ↓
NGINX CONTAINER
/etc/nginx/certs/
```

---

## 25.4 Nginx network

```yaml
networks:
  - frontend
```

Nginx is not attached to the backend network.

It can communicate with frontend services such as:

```text
wordpress
phpmyadmin
```

but it does not need direct access to MySQL.

---

# 26. WP-CLI service

```yaml
wpcli:
  image: wordpress:cli-php8.2
  profiles:
    - tools
  user: "33:33"
  working_dir: /var/www/html
  depends_on:
    - mysql
  environment:
    HOME: /tmp
    WORDPRESS_DB_HOST: mysql
    WORDPRESS_DB_NAME: "{{ mysql_database }}"
    WORDPRESS_DB_USER: "{{ mysql_user }}"
    WORDPRESS_DB_PASSWORD: "{{ mysql_password }}"
  volumes:
    - wordpress_data:/var/www/html
  networks:
    - backend
```

WP-CLI is used for automatic WordPress installation/checks.

It is not a web server and does not need to stay running.

---

## 26.1 Profile

```yaml
profiles:
  - tools
```

Normal:

```bash
docker compose up -d
```

will not permanently start this profile service.

When Ansible needs WP-CLI it explicitly runs:

```bash
docker compose --profile tools run --rm wpcli ...
```

---

## 26.2 `user: "33:33"`

The one-off WP-CLI container runs with numeric user/group ID:

```text
33:33
```

This matches the ownership expectations of the shared WordPress files in this setup and avoids WP-CLI creating shared files as root.

---

## 26.3 `working_dir`

```yaml
working_dir: /var/www/html
```

Commands run from the WordPress directory by default.

---

## 26.4 `HOME: /tmp`

```yaml
HOME: /tmp
```

Gives WP-CLI a writable home location in the temporary container.

---

## 26.5 Shared volume

```yaml
- wordpress_data:/var/www/html
```

The WP-CLI one-off container sees exactly the same WordPress files as the WordPress PHP-FPM service.

---

## 26.6 Backend network

```yaml
networks:
  - backend
```

WP-CLI needs database access but does not need to receive public web traffic.

---

# 27. Docker networks

The template defines:

```yaml
networks:
  frontend:

  backend:
    internal: true
```

The service connections are:

```text
                FRONTEND
    ┌─────────────────────────────┐
    │                             │
    │ nginx                       │
    │   │                         │
    │   ├──── wordpress           │
    │   │                         │
    │   └──── phpMyAdmin          │
    │                             │
    └────────────┬────────────────┘
                 │
                 │ services that also join backend
                 ▼
                 BACKEND
    ┌─────────────────────────────┐
    │                             │
    │ wordpress ────┐             │
    │               ├── MySQL     │
    │ phpMyAdmin ───┤             │
    │               │             │
    │ WP-CLI ───────┘             │
    │                             │
    └─────────────────────────────┘
```

MySQL is not on the frontend network.

Nginx is not on the backend network.

---

## 27.1 `internal: true`

```yaml
backend:
  internal: true
```

This marks the backend Docker network as internal.

The intention is to isolate backend-only service communication from external network access.

For this project, MySQL is also not published with a host port.

So users on the Internet do not connect directly using:

```text
74.248.37.10:3306
```

WordPress and phpMyAdmin reach MySQL through Docker networking instead.

---

# 28. Docker volumes

The template defines:

```yaml
volumes:
  mysql_data:
  wordpress_data:
```

These are named Docker volumes.

---

## 28.1 `mysql_data`

Stores the MySQL database files.

```text
mysql container
      ↓
/var/lib/mysql
      ↓
mysql_data named volume
```

---

## 28.2 `wordpress_data`

Stores WordPress files.

It is shared by:

```text
WordPress PHP-FPM → read/write
Nginx             → read-only
WP-CLI            → read/write
```

---

## 28.3 Why volumes matter

Containers are replaceable.

If a container is recreated, data should not disappear with its writable container layer.

```text
container recreated
        ↓
named volume remains
        ↓
site data/files remain available
```

Be careful with:

```bash
docker compose down -v
```

The `-v` explicitly removes named volumes associated with the Compose project and is therefore used only when you intentionally want a clean reset.

---

# 29. Nginx template

File:

```text
roles/wordpress/templates/nginx.conf.j2
```

The template changes depending on `use_letsencrypt`.

---

# 30. Nginx when Let's Encrypt is enabled

This conditional block exists only when TLS is true:

```jinja
{% if use_letsencrypt %}
server {
    listen 80;
    server_name {{ domain_name }};

    return 301 https://$host$request_uri;
}
{% endif %}
```

So with TLS enabled:

```text
Browser requests HTTP :80
        ↓
Nginx receives request
        ↓
301 redirect
        ↓
https://same-host/same-path
```

### `$host`

The host requested by the browser.

### `$request_uri`

The requested path and query string.

Example:

```text
http://abel-hid.duckdns.org/test?a=1
```

becomes approximately:

```text
https://abel-hid.duckdns.org/test?a=1
```

---

# 31. Main Nginx server: conditional listen

The next server block contains:

```jinja
{% if use_letsencrypt %}
    listen 443 ssl;
{% else %}
    listen 80;
{% endif %}
```

Therefore:

```text
True  → listen 443 ssl
False → listen 80
```

This is why the same Nginx template supports both modes without self-signed TLS.

---

# 32. `server_name`

```nginx
server_name {{ domain_name }};
```

With the current inventory this becomes:

```nginx
server_name abel-hid.duckdns.org;
```

If you test HTTP mode using the IP as `domain_name`, it can become:

```nginx
server_name 74.248.37.10;
```

---

# 33. TLS certificate configuration

Only when Let's Encrypt is enabled:

```jinja
{% if use_letsencrypt %}
    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
{% endif %}
```

These paths are **inside the Nginx container**.

The Compose file maps the host directory:

```text
/opt/cloud1/certs
```

into:

```text
/etc/nginx/certs
```

So:

```text
HOST
/opt/cloud1/certs/fullchain.pem
/opt/cloud1/certs/privkey.pem
        ↓ bind mount
NGINX CONTAINER
/etc/nginx/certs/fullchain.pem
/etc/nginx/certs/privkey.pem
```

---

# 34. WordPress document root

```nginx
root /var/www/html;
index index.php index.html;
```

Nginx sees the shared WordPress volume at:

```text
/var/www/html
```

The default index files are:

```text
index.php
index.html
```

---

# 35. `/phpmyadmin` redirect

```nginx
location = /phpmyadmin {
    return 301 $scheme://$host/phpmyadmin/;
}
```

This handles exactly:

```text
/phpmyadmin
```

and redirects it to:

```text
/phpmyadmin/
```

The trailing slash makes the reverse-proxy path behavior consistent.

### `$scheme`

Current request scheme:

```text
http
```

or:

```text
https
```

---

# 36. phpMyAdmin reverse proxy

```nginx
location ^~ /phpmyadmin/ {
    proxy_pass http://phpmyadmin:80/;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Requests such as:

```text
https://abel-hid.duckdns.org/phpmyadmin/
```

arrive at Nginx.

Nginx sends them internally to:

```text
http://phpmyadmin:80/
```

`phpmyadmin` is the Docker Compose service name, resolved through Docker DNS on the frontend network.

### `proxy_set_header Host $host`

Pass the original requested host to phpMyAdmin.

### `X-Forwarded-Proto`

Tells the proxied application whether the original browser request used HTTP or HTTPS.

---

# 37. WordPress route handling

```nginx
location / {
    try_files $uri $uri/ /index.php?$args;
}
```

Nginx checks in order:

```text
Does requested file exist?
        ↓
$uri

Does requested directory exist?
        ↓
$uri/

Otherwise
        ↓
/index.php?$args
```

The last fallback sends WordPress-style routes through `index.php`.

For example a pretty URL can still be handled by WordPress even if there is no physical directory with that name.

---

# 38. PHP FastCGI handling

```nginx
location ~ \.php$ {
    fastcgi_pass wordpress:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param HTTPS $https if_not_empty;
    include fastcgi_params;
}
```

Nginx itself does not execute PHP.

It sends PHP requests to:

```text
wordpress:9000
```

where:

```text
wordpress
```

is the Compose service name and port `9000` is the PHP-FPM service port inside Docker networking.

Conceptually:

```text
Browser requests index.php
        ↓
Nginx
        ↓ FastCGI
wordpress PHP-FPM :9000
        ↓
PHP executes WordPress
        ↓
response returns to Nginx
        ↓
Browser
```

---

# 39. Complete request flow for WordPress

With HTTPS enabled:

```text
Browser
  |
  | https://abel-hid.duckdns.org/
  v
Azure public IP :443
  |
  v
Nginx container
  |
  | PHP request
  v
wordpress:9000
  |
  | database query
  v
mysql:3306 (Docker backend network only)
```

The response travels back in reverse:

```text
MySQL
  ↓
WordPress PHP-FPM
  ↓
Nginx
  ↓
Browser
```

---

# 40. Complete request flow for phpMyAdmin

```text
Browser
  |
  | /phpmyadmin/
  v
Nginx
  |
  | reverse proxy
  v
phpmyadmin:80
  |
  | database connection
  v
mysql
```

The browser never connects directly to MySQL.

---

# 41. Why MySQL may show `3306/tcp` in Docker output

A container can expose/listen on a port internally without that port being published on the host.

Compare:

```text
3306/tcp
```

with:

```text
0.0.0.0:3306->3306/tcp
```

The first means a port exists in the container/network context.

The second would indicate host publishing.

Your MySQL service has no Compose `ports:` entry, so it is not published like Nginx ports 80/443.

---

# 42. `restart: unless-stopped` + systemd + volumes

These three pieces solve different parts of reboot persistence.

```text
Docker systemd enabled
        ↓
Docker starts after Ubuntu reboot

restart: unless-stopped
        ↓
long-running containers start again

named volumes
        ↓
MySQL and WordPress persistent data remain
```

Together:

```text
VM reboots
   ↓
Docker starts
   ↓
containers restart
   ↓
volumes are reattached
   ↓
site returns with its data
```

---

# 43. What is idempotency?

Idempotency means you can apply the deployment repeatedly and still end in the same desired working state.

Conceptually:

```text
run playbook
run playbook again
run playbook again
        ↓
working site remains the same desired site
```

Examples in the final code:

```text
apt state: present
    ↓
package installed only if needed

file state: directory
    ↓
directory created only if missing

stat certificate
    ↓
Certbot runs only when certificate is missing

wp core is-installed
    ↓
WordPress installation only runs when needed

Docker named volumes
    ↓
data is reused on later runs
```

The second run can still contain some command tasks, but it must not break or duplicate the deployment.

---

# 44. `ok`, `changed`, `skipping`, `failed`

Ansible output uses these statuses.

## `ok`

```text
ok: [cloud1-azure]
```

The task succeeded and Ansible does not report a change.

## `changed`

```text
changed: [cloud1-azure]
```

The task succeeded and changed something.

Normal on a first deployment.

## `skipping`

```text
skipping: [cloud1-azure]
```

A `when:` condition evaluated false.

For example TLS tasks are skipped when:

```text
use_letsencrypt=False
```

## `failed`

```text
FAILED
```

Ansible reached the host but a task failed.

## `unreachable`

Ansible could not communicate with the host, often because of SSH/address/authentication/network problems.

---

# 45. Why the current project has both HTTP and HTTPS modes

The same Ansible code can render two configurations.

## HTTPS/domain mode

Inventory:

```ini
domain_name=abel-hid.duckdns.org use_letsencrypt=True
```

Flow:

```text
Certbot
  ↓
Let's Encrypt certificate
  ↓
Compose exposes 80 + 443
  ↓
HTTP redirects to HTTPS
  ↓
WordPress URL = https://abel-hid.duckdns.org
```

## HTTP mode

Example inventory values:

```ini
domain_name=74.248.37.10 use_letsencrypt=False
```

Flow:

```text
TLS tasks skipped
  ↓
Compose exposes only 80
  ↓
Nginx listens on 80
  ↓
WordPress URL = http://74.248.37.10
```

There is no self-signed branch.

---

# 46. Same-VM switch from false to true

The final TLS role contains support for this situation.

Suppose you already deployed:

```text
use_letsencrypt=False
```

Nginx is running on port 80.

Then you change to:

```text
use_letsencrypt=True
```

The TLS role checks:

```text
Does certificate already exist?
Does /opt/cloud1/docker-compose.yml already exist?
```

If:

```text
certificate missing
AND
old Compose deployment exists
```

it runs:

```bash
docker compose stop nginx
```

so Certbot standalone can temporarily use port 80.

Then Certbot obtains the certificate.

Later the WordPress role renders the HTTPS Compose/Nginx files and starts Nginx again.

---

# 47. `docker compose up -d` and individual restart

The complete stack is launched in one command:

```bash
docker compose up -d
```

An individual service can also be restarted, for example:

```bash
docker compose restart nginx
```

The handler uses exactly that command for Nginx.

---

# 48. Where files live: local versus remote

This distinction is essential.

## Local source files

```text
ansible/
├── roles/wordpress/templates/docker-compose.yml.j2
└── roles/wordpress/templates/nginx.conf.j2
```

You edit these.

## Remote generated files

```text
/opt/cloud1/docker-compose.yml
/opt/cloud1/nginx.conf
```

Ansible generates these.

When TLS is enabled:

```text
/opt/cloud1/certs/fullchain.pem
/opt/cloud1/certs/privkey.pem
```

are also prepared on the remote VM.

Do not think of `/opt/cloud1/docker-compose.yml` as the source of truth.

The Ansible template is the source of truth because the next Ansible run can render it again.

---

# 49. Testing Ansible connectivity

From inside the local `ansible/` directory:

```bash
ansible-inventory --graph
```

This verifies that Ansible can parse the inventory.

You should see the `webserver` group and `cloud1-azure` host.

Then:

```bash
ansible webserver -m ping \
  --vault-password-file ~/Desktop/PASS.txt
```

Ansible's `ping` module is not normal ICMP ping.

It checks the management path roughly like:

```text
Can Ansible resolve the inventory host?
        ↓
Can it SSH using the configured user/key?
        ↓
Can it run Python remotely?
        ↓
Can Ansible communicate successfully?
```

Successful output includes:

```text
"ping": "pong"
```

---

# 50. Running the full deployment

From the `ansible/` directory:

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Desktop/PASS.txt
```

High-level flow:

```text
Load ansible.cfg
      ↓
Load inventory.ini
      ↓
Load group variables
      ↓
Decrypt vault.yml using PASS.txt
      ↓
SSH to cloud1-azure
      ↓
Gather Ansible facts
      ↓
become root remotely
      ↓
Docker role
      ↓
Firewall role
      ↓
TLS role
      ↓
WordPress role
      ↓
Handlers if notified
```

---

# 51. What does `Gathering Facts` mean?

At the beginning you see:

```text
TASK [Gathering Facts]
```

Ansible collects information from the remote system.

The project later uses one of these facts:

```jinja
{{ ansible_distribution_release }}
```

for Docker's Ubuntu repository configuration.

So fact gathering is not random output; the Docker role uses remote OS information.

---

# 52. First run versus second run

## First run

Many things do not exist yet:

```text
Docker packages
firewall rules
/opt/cloud1
certificate
containers
WordPress database installation
```

So many tasks may show:

```text
changed
```

## Second run

Run exactly the same playbook again:

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Desktop/PASS.txt
```

Now:

```text
packages already installed
directories already exist
certificate already exists
containers already exist
WordPress already installed
```

So more tasks should be `ok` or `skipping`, and most importantly:

```text
failed=0
unreachable=0
```

and the site remains working.

---

# 53. Test the HTTPS mode

Current inventory uses:

```ini
domain_name=abel-hid.duckdns.org use_letsencrypt=True
```

After deployment:

```bash
curl -I http://abel-hid.duckdns.org
```

should show HTTP redirect behavior.

Then:

```bash
curl -I https://abel-hid.duckdns.org
```

should reach the HTTPS site if DNS, cloud firewall/network rules, and certificate issuance are all correct.

You can also inspect:

```bash
sudo docker compose -f /opt/cloud1/docker-compose.yml ps
```

or after entering the directory:

```bash
cd /opt/cloud1
sudo docker compose ps
```

---

# 54. Test the HTTP-only mode on the same VM

If you intentionally reset the application and change inventory to:

```ini
domain_name=74.248.37.10 use_letsencrypt=False
```

then deploy again.

Expected generated behavior:

```text
TLS tasks: skipped
Nginx host port 80: published
Nginx host port 443: not published by Compose
Nginx config: listen 80
WordPress URL: http://74.248.37.10
```

Test:

```bash
curl -I http://74.248.37.10
```

A normal successful WordPress response can look like:

```text
HTTP/1.1 200 OK
```

Testing:

```bash
curl -kI https://74.248.37.10
```

can fail to connect because the false branch does not create a TLS listener.

That is expected for this code.

---

# 55. Check public listening ports

On the VM:

```bash
sudo ss -lntp
```

In HTTP-only mode, relevant public host listeners should include:

```text
22
80
```

In Let's Encrypt mode they can include:

```text
22
80
443
```

MySQL should not be published from Compose as a host `3306` mapping.

---

# 56. Inspect generated Compose file

On the VM:

```bash
sudo cat /opt/cloud1/docker-compose.yml
```

Remember: this file contains rendered secret values, so do not casually paste it publicly.

For easier line-number inspection:

```bash
sudo nl -ba /opt/cloud1/docker-compose.yml
```

This is useful when Docker Compose reports a YAML line number.

---

# 57. Inspect generated Nginx file

```bash
sudo cat /opt/cloud1/nginx.conf
```

### With TLS true, look for

```nginx
listen 80;
return 301 https://$host$request_uri;
```

and:

```nginx
listen 443 ssl;
ssl_certificate ...;
ssl_certificate_key ...;
```

### With TLS false, look for

```nginx
listen 80;
```

and there should be no active TLS certificate directives generated from the conditional block.

---

# 58. Check containers

```bash
cd /opt/cloud1
sudo docker compose ps
```

The long-running services are:

```text
mysql
wordpress
phpmyadmin
nginx
```

`wpcli` is a one-off tool service and is not expected to remain running after `docker compose run --rm` finishes.

---

# 59. Why WordPress can connect to MySQL without public port 3306

Docker networking is separate from host port publishing.

```text
Internet
   X
   │ no direct MySQL host mapping
   │
MySQL
```

But internally:

```text
WordPress container
       │
       │ backend Docker network
       ▼
mysql:3306
```

So database communication works without exposing the database to Internet clients.

---

# 60. Why phpMyAdmin does not need its own public port

There is no Compose host mapping such as:

```text
8080:80
```

for phpMyAdmin.

Instead:

```text
Browser
   ↓
Nginx public 80/443
   ↓
/phpmyadmin/
   ↓
reverse proxy
   ↓
phpmyadmin:80 inside Docker
```

This keeps Nginx as the single web entry point.

---

# 61. Why WordPress PHP-FPM does not need a public port

Nginx talks to it internally:

```nginx
fastcgi_pass wordpress:9000;
```

So there is no reason for the browser to connect directly to PHP-FPM.

The browser only talks to Nginx.

---

# 62. The most important variables and where they go

```text
inventory.ini
────────────────────────────────
ansible_host
    → SSH destination

ansible_user
    → SSH login user

ansible_ssh_private_key_file
    → SSH authentication key

domain_name
    → Certbot
    → Nginx
    → WordPress URL
    → phpMyAdmin URL

use_letsencrypt
    → TLS tasks
    → site_scheme
    → Compose port 443
    → certificate mount
    → Nginx HTTP/HTTPS configuration
```

```text
vars.yaml
────────────────────────────────
app_dir
    → /opt/cloud1 paths

site_scheme
    → http or https

mysql_database
    → MySQL + WordPress + WP-CLI

mysql_user
    → MySQL + WordPress + WP-CLI

mysql_root_password
    → MySQL root initialization/readiness check

mysql_password
    → MySQL + WordPress + WP-CLI

wordpress_title
wordpress_admin_user
wordpress_admin_email
wordpress_admin_password
    → wp core install
```

---

# 63. The secret flow

```text
Encrypted vault.yml
       │
       ├── vault_mysql_root_password
       ├── vault_mysql_password
       └── vault_wordpress_admin_password
       │
       ▼
vars.yaml references
       │
       ├── mysql_root_password
       ├── mysql_password
       └── wordpress_admin_password
       │
       ▼
Ansible renders/executes required tasks
       │
       ├── MySQL environment
       ├── WordPress DB environment
       ├── WP-CLI DB environment
       └── wp core install admin password
```

`vault.yml` remains encrypted in the source project.

Ansible decrypts the values at execution time using the external Vault password.

---

# 64. Three different kinds of “variables” you see

It helps to distinguish them.

## Ansible/Jinja variables

```jinja
{{ app_dir }}
{{ domain_name }}
{{ mysql_password }}
```

Evaluated by Ansible while running/rendering.

## Docker Compose environment variables

Example rendered keys:

```text
MYSQL_ROOT_PASSWORD
WORDPRESS_DB_HOST
PMA_HOST
```

Passed into containers as environment variables.

## Shell environment variable inside MySQL container

```text
$MYSQL_ROOT_PASSWORD
```

This is expanded by:

```text
sh -c
```

inside the running MySQL container.

These are different layers.

---

# 65. Why `wordpress_data` is mounted read-only in Nginx

Nginx only needs to read static/site files.

The template uses:

```yaml
wordpress_data:/var/www/html:ro
```

The `:ro` means Nginx cannot modify the shared WordPress volume through that mount.

WordPress PHP-FPM and WP-CLI use the volume without `:ro`, because they may need to write WordPress files.

---

# 66. Why Nginx has no backend network

Nginx needs to reach:

```text
WordPress PHP-FPM
phpMyAdmin
```

Both are on `frontend`.

Nginx does not need to connect directly to MySQL.

Therefore the final template only attaches Nginx to:

```yaml
networks:
  - frontend
```

This keeps responsibilities separated.

---

# 67. Why MySQL has no frontend network

MySQL only needs internal database clients:

```text
WordPress
phpMyAdmin
WP-CLI
```

All of them share the `backend` network with MySQL.

So MySQL does not join `frontend` and does not receive a public host port.

---

# 68. Why WordPress and phpMyAdmin have two networks

They act as bridges between layers.

WordPress:

```text
frontend → receives FastCGI traffic from Nginx
backend  → queries MySQL
```

phpMyAdmin:

```text
frontend → receives reverse-proxy traffic from Nginx
backend  → connects to MySQL
```

---

# 69. Why there is a WP-CLI container at all

Without WP-CLI, a brand-new WordPress installation normally needs browser interaction.

The final automation wants the initial site to be created automatically.

WP-CLI lets Ansible execute:

```text
wp core is-installed
wp core install
```

So deployment can finish with an initialized WordPress site without manually completing the setup screen.

---

# 70. Why WP-CLI is not a permanent service

WP-CLI is a command-line tool, not a daemon/server.

The project invokes it using:

```bash
docker compose --profile tools run --rm wpcli ...
```

Lifecycle:

```text
create temporary wpcli container
        ↓
run WP command
        ↓
command finishes
        ↓
--rm deletes temporary container
```

The data persists because the WordPress files are in `wordpress_data` and the site state is also stored in MySQL.

---

# 71. What happens after an Ubuntu reboot?

```text
Ubuntu boots
    ↓
Docker systemd unit is enabled
    ↓
Docker starts
    ↓
restart: unless-stopped policies apply
    ↓
mysql / wordpress / phpmyadmin / nginx start
    ↓
named volumes reattach
    ↓
site data remains
```

You do not need to rerun `wp core install` after every reboot because installation state is persisted in the database.

---

# 72. What happens if you rerun Ansible after the site already works?

High-level behavior:

```text
Docker packages already present
      ↓
mostly ok

UFW rules already present
      ↓
no duplicate functional rules

Certificate already exists
      ↓
Certbot obtain task skipped

Templates same
      ↓
no unnecessary Nginx handler from unchanged template

Containers already exist
      ↓
Compose reconciles desired state

MySQL/WordPress readiness checks
      ↓
pass

wp core is-installed
      ↓
rc = 0
      ↓
initial installation skipped
```

This is the core idempotent behavior you should be able to explain.

---

# 73. Useful defense explanations in one sentence

## What is Ansible?

> Ansible is the automation tool running on my local machine; it connects to the Ubuntu server over SSH and applies the infrastructure described by my playbook and roles.

## What is the inventory?

> The inventory defines the target host, its SSH connection information, and host-specific variables such as the domain and whether Let's Encrypt is enabled.

## What is a role?

> A role groups related Ansible tasks, so Docker installation, firewall configuration, TLS and application deployment are separated logically.

## What is `become: true`?

> It tells Ansible to use privilege escalation on the remote machine because package installation, firewall configuration, Docker administration and system paths require root privileges.

## What is Ansible Vault?

> Vault encrypts sensitive variables such as MySQL and WordPress passwords so the source file does not contain them in plaintext.

## What is idempotency?

> Running the deployment repeatedly should converge to the same working state without reinstalling WordPress or recreating things unnecessarily.

## Why Docker Compose?

> Compose declares all the containers, networks and volumes in one file and lets the whole stack be started together while still allowing individual services to be managed.

## Why Nginx?

> Nginx is the single public web entry point; it serves/routes WordPress PHP traffic and reverse-proxies `/phpmyadmin/`.

## Why isn't MySQL public?

> MySQL has no host port mapping and is only attached to the internal backend Docker network, so application containers reach it through Docker networking instead of exposing it to the Internet.

## Why two Docker networks?

> The frontend connects Nginx to web-facing application services, while the internal backend connects WordPress, phpMyAdmin and WP-CLI to MySQL without putting MySQL on the frontend network.

## Why volumes?

> Named volumes keep the database and WordPress files persistent when containers are recreated or the server reboots.

## Why WP-CLI?

> WP-CLI automates the initial WordPress installation so the deployment does not depend on manually completing the browser setup form.

## What does `use_letsencrypt` do?

> It switches the generated deployment between Let's Encrypt HTTPS mode and HTTP-only mode; there is no self-signed certificate branch in the final code.

## Why does false mode have no HTTPS?

> When `use_letsencrypt=False`, TLS tasks are skipped, port 443 is not published in the Compose template, and Nginx listens on port 80 only.

## Why does true mode still expose port 80?

> Port 80 is used by Nginx to redirect normal HTTP requests to HTTPS, and Certbot standalone also needs port 80 when obtaining a new certificate.

---

# 74. Final complete flow

```text
                         LOCAL MACHINE
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│ ansible.cfg                                                  │
│    │                                                         │
│    └── inventory = ./inventory.ini                           │
│                                                              │
│ inventory.ini                                                │
│    │                                                         │
│    ├── 74.248.37.10                                         │
│    ├── azureuser                                            │
│    ├── SSH private key                                      │
│    ├── abel-hid.duckdns.org                                 │
│    └── use_letsencrypt=True                                 │
│                                                              │
│ group_vars/all/                                             │
│    │                                                         │
│    ├── vars.yaml                                            │
│    └── vault.yml 🔒                                         │
│                                                              │
│ site.yml                                                     │
│    │                                                         │
│    ├── docker role                                          │
│    ├── firewall role                                        │
│    ├── tls role                                             │
│    └── wordpress role                                       │
│                                                              │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                │ SSH :22
                                ▼
                         REMOTE UBUNTU VM
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│ Docker Engine                                                │
│                                                              │
│                    INTERNET                                  │
│                       │                                      │
│              HTTP 80 / HTTPS 443                            │
│                       │                                      │
│                       ▼                                      │
│                  ┌─────────┐                                 │
│                  │  NGINX  │                                 │
│                  └────┬────┘                                 │
│                      / \                                     │
│                     /   \                                    │
│                    ▼     ▼                                   │
│           ┌──────────┐ ┌──────────┐                          │
│           │WordPress │ │phpMyAdmin│                          │
│           │ PHP-FPM  │ │          │                          │
│           └────┬─────┘ └────┬─────┘                          │
│                │            │                                │
│                └──────┬─────┘                                │
│                       ▼                                      │
│                  ┌─────────┐                                 │
│                  │  MySQL  │                                 │
│                  └─────────┘                                 │
│                                                              │
│ Named volumes:                                               │
│   mysql_data                                                 │
│   wordpress_data                                             │
│                                                              │
│ Temporary tool when needed:                                  │
│   WP-CLI                                                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

# 75. The deployment in one sequence

```text
ansible-playbook site.yml --vault-password-file ~/Desktop/PASS.txt
        ↓
Ansible loads inventory
        ↓
Ansible decrypts Vault
        ↓
SSH to Azure
        ↓
Gather facts
        ↓
become root
        ↓
Install/configure Docker
        ↓
Configure UFW
        ↓
If use_letsencrypt=True:
    prepare Let's Encrypt certificate
Else:
    skip TLS tasks
        ↓
Render docker-compose.yml
        ↓
Render nginx.conf for HTTP or HTTPS mode
        ↓
docker compose up -d
        ↓
Wait for MySQL
        ↓
Wait for WordPress files
        ↓
Check WordPress installation
        ↓
Install with WP-CLI only if missing
        ↓
Working automated WordPress stack
```

That is the current final code from beginning to end.
