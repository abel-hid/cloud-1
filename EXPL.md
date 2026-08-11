Yes. Before continuing the deployment, you should understand **what every part is doing**, because in the 42 evaluation they can stop and ask things like “What is inventory?”, “Why Vault?”, “What does `become` do?”, “Why do you need roles?”, “What is idempotency?”, or “How does WordPress reach MySQL?”

The subject itself says you must be able to understand and defend the technical decisions, not just have AI-generated code that happens to work.

I’ll explain this from zero using **your actual Cloud-1 project**, not generic Ansible theory.

---

# 1. First understand the whole project

You have two machines involved.

```text
YOUR 42 / LOCAL MACHINE
        |
        | SSH
        | Ansible commands
        v
AZURE UBUNTU SERVER
        |
        | Docker
        v
+------------------------------+
| nginx                        |
| wordpress PHP-FPM            |
| phpMyAdmin                   |
| MySQL                        |
+------------------------------+
```

Your **local machine** is where you have:

```text
cloud-1-test/
└── ansible/
    ├── ansible.cfg
    ├── inventory.ini
    ├── site.yml
    ├── group_vars/
    └── roles/
```

You execute:

```bash
ansible-playbook site.yml --ask-vault-pass
```

on your **local machine**.

Ansible then connects through SSH to:

```text
74.248.37.10
```

and configures the **remote Azure server**.

This distinction is extremely important.

You are NOT manually doing:

```bash
ssh server
sudo apt install docker
sudo docker ...
```

for the deployment.

Instead, you tell Ansible:

> “Make this server have Docker, firewall rules, TLS, WordPress, MySQL, Nginx, etc.”

Ansible performs those operations remotely.

That is the main point of Cloud-1: automated infrastructure deployment. The subject explicitly asks for deployment automation, several separate containers, persistence, networking, TLS when possible, and the ability to reproduce the deployment on fresh Ubuntu-like machines.

---

# 2. What is Ansible?

Think of Ansible as a remote system administrator.

Without Ansible, you might write:

```bash
ssh azureuser@74.248.37.10
```

then manually do:

```bash
sudo apt update
sudo apt install docker
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
...
```

With Ansible, you describe those operations in YAML files:

```yaml
- name: Install UFW
  ansible.builtin.apt:
    name: ufw
    state: present
```

Ansible connects to the server and does it for you.

The important idea is:

```text
Ansible is installed locally
         ↓
Ansible reads your YAML
         ↓
Ansible connects through SSH
         ↓
Ansible changes the remote server
```

Ansible normally does **not need an agent installed on the server**.

Your subject assumes the target server starts basically with:

```text
Ubuntu 22.04-like OS
SSH
Python
```

Then your automation handles the rest.

---

# 3. What is `(.ansible-venv)`?

You see this:

```text
(.ansible-venv) ➜ cloud-1-test
```

The:

```text
(.ansible-venv)
```

means you activated a **Python virtual environment**.

A virtual environment gives you a private Python environment where you can install packages such as Ansible without needing:

```bash
sudo pip install ...
```

That matters on a 42 machine because you don't have system-level sudo permissions.

Conceptually:

```text
System Python
    |
    +-- your normal machine packages

.ansible-venv/
    |
    +-- its own Python packages
    +-- ansible
    +-- ansible-playbook
```

You can check:

```bash
which ansible
```

You will probably see something pointing inside:

```text
.ansible-venv/bin/ansible
```

You can check the version:

```bash
ansible --version
```

---

# 4. Understand your terminal prompt

You showed:

```text
(.ansible-venv) ➜  cloud-1-test ansible webserver -m ping
```

Be careful: there are **two different `ansible` words that can appear**, but one may just be your folder name.

For example:

```text
(.ansible-venv) ➜ ansible ansible-playbook site.yml
                   ^       ^
                   |       |
              folder name  command
```

The first:

```text
ansible
```

after the arrow is your **current directory** shown by zsh.

The actual command begins after it:

```bash
ansible-playbook site.yml
```

Same thing here:

```text
➜ cloud-1-test ansible webserver -m ping
   ^^^^^^^^^^^^ ^^^^^^^
   folder       command
```

So don't copy the prompt itself.

You type only:

```bash
ansible webserver -m ping --ask-vault-pass
```

---

# 5. Your project structure

Your project is approximately:

```text
cloud-1-test/
└── ansible/
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
        │   └── tasks/main.yml
        │
        ├── firewall/
        │   └── tasks/main.yml
        │
        ├── tls/
        │   └── tasks/main.yml
        │
        └── wordpress/
            ├── tasks/main.yml
            ├── handlers/main.yml
            └── templates/
                ├── docker-compose.yml.j2
                └── nginx.conf.j2
```

This separation is intentional because the subject asks for Ansible to be organized into relevant roles.

Now let's understand every one.

---

# 6. `ansible.cfg`

Your file is:

```text
ansible/ansible.cfg
```

Think of this as:

> configuration for Ansible itself.

For example:

```ini
[defaults]

inventory = ./inventory.ini
```

This tells Ansible:

> “Unless I tell you otherwise, use `inventory.ini` to know which servers exist.”

That is why this matters:

```bash
cd cloud-1-test/ansible
```

Then:

```bash
ansible webserver ...
```

Because Ansible looks for its configuration starting from places including your current working directory.

When you were here:

```text
cloud-1-test/
```

Ansible did not find:

```text
cloud-1-test/ansible.cfg
```

because the actual file was:

```text
cloud-1-test/ansible/ansible.cfg
```

So it didn't automatically know:

```text
inventory = ./inventory.ini
```

That caused:

```text
No inventory was parsed
```

---

# 7. What is an inventory?

The inventory answers:

> “Which machines does Ansible control?”

Your:

```text
inventory.ini
```

contains something like:

```ini
[webserver]

cloud1-azure ansible_host=74.248.37.10 ansible_user=azureuser ansible_ssh_private_key_file=~/.ssh/id_rsa domain_name=abel-hid.duckdns.org use_letsencrypt=True
```

Let's break this apart.

---

## `[webserver]`

This:

```ini
[webserver]
```

creates a **group of machines** called:

```text
webserver
```

You could imagine:

```ini
[webserver]
server1 ...
server2 ...
server3 ...
```

Then Ansible could deploy to all three.

That is related directly to the subject requirement that deployment should be possible on several servers in parallel.

---

# 8. What is `cloud1-azure`?

Here:

```ini
cloud1-azure ansible_host=74.248.37.10 ...
```

`cloud1-azure` is an **Ansible inventory name**.

It's basically our friendly name for the machine.

Instead of constantly saying:

```text
74.248.37.10
```

Ansible can refer to it as:

```text
cloud1-azure
```

That's why your logs say:

```text
ok: [cloud1-azure]
```

It doesn't mean the real Linux hostname necessarily has to be `cloud1-azure`.

It's the name we gave the host in Ansible.

---

# 9. `ansible_host`

Then:

```ini
ansible_host=74.248.37.10
```

means:

> “The real address I should SSH into is `74.248.37.10`.”

So:

```text
cloud1-azure
      ↓
74.248.37.10
```

---

# 10. `ansible_user`

You have:

```ini
ansible_user=azureuser
```

That means Ansible connects as:

```text
azureuser
```

Equivalent conceptually to:

```bash
ssh azureuser@74.248.37.10
```

---

# 11. `ansible_ssh_private_key_file`

You have:

```ini
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

This tells Ansible:

> “Use this SSH private key to authenticate.”

Equivalent conceptually to:

```bash
ssh -i ~/.ssh/id_rsa azureuser@74.248.37.10
```

So Ansible is doing SSH for you.

---

# 12. `domain_name`

You have:

```ini
domain_name=abel-hid.duckdns.org
```

That's a custom variable we created.

It is later used for things such as:

```text
WordPress URL
Nginx server_name
Let's Encrypt certificate
phpMyAdmin URL
```

So changing this one variable can change the domain used across the deployment.

That's one advantage of automation: don't hard-code the domain in fifteen files.

---

# 13. `use_letsencrypt=True`

You have:

```ini
use_letsencrypt=True
```

That's another custom variable.

It tells the TLS role:

```text
Do I have a real public domain?
        |
       YES
        |
        v
Use Let's Encrypt
```

Without it:

```text
No usable public DNS
        |
        v
generate self-signed TLS certificate
```

Your server has:

```text
abel-hid.duckdns.org
```

so we use Let's Encrypt.

---

# 14. Now understand your `ansible webserver -m ping`

Your command is:

```bash
ansible webserver -m ping --ask-vault-pass
```

Break it apart.

```text
ansible
```

is the program.

Then:

```text
webserver
```

means:

> “Run this against every machine belonging to the `[webserver]` inventory group.”

Then:

```text
-m ping
```

means:

> “Use the Ansible module called `ping`.”

`-m` means:

```text
module
```

So:

```bash
ansible webserver -m ping
```

means roughly:

> “Ansible, execute your ping module against my webserver group.”

---

# 15. Ansible `ping` is NOT normal network ping

This is very important.

This:

```bash
ping 74.248.37.10
```

is ICMP/network ping.

But:

```bash
ansible webserver -m ping
```

is different.

Ansible effectively verifies:

```text
Can I find this host?
        ↓
Can I SSH into it?
        ↓
Can I execute Python remotely?
        ↓
Can Ansible communicate with it?
```

Successful result:

```text
cloud1-azure | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

That means Ansible is ready to manage the server.

---

# 16. Why did you get “No inventory was parsed”?

You ran:

```text
➜ cloud-1-test
```

instead of:

```text
➜ ansible
```

Your configuration lives here:

```text
cloud-1-test/
└── ansible/
    └── ansible.cfg
```

So from:

```text
cloud-1-test/
```

Ansible didn't automatically load that config.

Without the configuration, it didn't automatically discover:

```text
inventory.ini
```

Therefore:

```text
Ansible
   ↓
Where is inventory?
   ↓
I don't know
   ↓
No hosts
   ↓
What is "webserver"?
   ↓
Could not match supplied host pattern
```

That explains exactly:

```text
[WARNING]: No inventory was parsed
[WARNING]: provided hosts list is empty
[WARNING]: Could not match supplied host pattern, ignoring: webserver
```

Nothing was wrong with Azure.

Nothing was wrong with SSH.

You were simply executing Ansible from the wrong project level.

---

# 17. Why this works

Do:

```bash
cd cloud-1-test/ansible
```

Now you're here:

```text
cloud-1-test/
└── ansible/       ← YOU ARE HERE
    ├── ansible.cfg
    ├── inventory.ini
    └── site.yml
```

Now:

```bash
ansible webserver -m ping --ask-vault-pass
```

Ansible sees:

```text
ansible.cfg
       ↓
inventory = ./inventory.ini
       ↓
[webserver]
       ↓
cloud1-azure
       ↓
74.248.37.10
```

Now it knows where to connect.

---

# 18. What is `--ask-vault-pass`?

Now we reach your second error:

```text
ERROR! Attempting to decrypt but no vault secrets found
```

To understand that, first understand **Ansible Vault**.

Your project needs passwords.

For example:

```text
MySQL root password
WordPress database password
WordPress administrator password
```

We absolutely do not want:

```yaml
mysql_password: mypassword123
```

sitting in Git.

The subject explicitly says that secrets must not be hard-coded.

So we have:

```text
group_vars/all/vault.yml
```

encrypted using Ansible Vault.

If you do:

```bash
cat group_vars/all/vault.yml
```

you should see something like:

```text
$ANSIBLE_VAULT;1.1;AES256
6631633734...
6261353234...
...
```

Not:

```yaml
password: hello123
```

---

# 19. Think of Vault like a locked box

Conceptually:

```text
vault.yml
+--------------------------------+
| 🔒 encrypted                   |
|                                |
| MySQL root password            |
| MySQL user password            |
| WordPress admin password       |
+--------------------------------+
              ^
              |
        Vault password
```

Ansible needs the Vault password to open that file.

That's what:

```bash
--ask-vault-pass
```

means:

> “Ask me interactively for the password needed to decrypt Ansible Vault.”

Therefore:

```bash
ansible webserver -m ping --ask-vault-pass
```

asks:

```text
Vault password:
```

You type it.

Your typing will normally **not appear on the screen**.

That's normal for password prompts.

Press Enter.

---

# 20. “But why does ping need my WordPress password?”

Excellent question.

You might think:

> “I'm only testing SSH. Why does Ansible care about WordPress secrets?”

Because Ansible automatically loads variables associated with the hosts.

You have:

```text
group_vars/
└── all/
    ├── vars.yaml
    └── vault.yml
```

`all` means:

> variables applying to all hosts.

When Ansible loads:

```text
cloud1-azure
```

it also loads:

```text
group_vars/all/
```

And because:

```text
vault.yml
```

is encrypted, Ansible needs the Vault password to parse the variables.

So even though the `ping` module itself doesn't need a MySQL password, the inventory variable-loading stage encounters the encrypted file.

Hence:

```bash
ansible webserver -m ping
```

can produce:

```text
Attempting to decrypt but no vault secrets found
```

while:

```bash
ansible webserver -m ping --ask-vault-pass
```

works.

---

# 21. `vars.yaml` versus `vault.yml`

This is an important design.

Normal configuration goes into:

```text
vars.yaml
```

For example:

```yaml
mysql_database: wordpress
mysql_user: wp_user
wordpress_title: Cloud-1
wordpress_admin_user: cloud1_admin
```

These values aren't secrets.

But passwords go into:

```text
vault.yml
```

Encrypted.

Conceptually:

```text
vars.yaml
-------------------------
database name
database username
domain configuration
WordPress title
WordPress username

vault.yml
-------------------------
PASSWORDS
SECRETS
```

And then `vars.yaml` can reference secret variables:

```yaml
mysql_root_password: "{{ vault_mysql_root_password }}"
```

Read that syntax as:

> “The value of `mysql_root_password` should be whatever is stored inside variable `vault_mysql_root_password`.”

---

# 22. What is `{{ ... }}`?

You'll see this everywhere:

```yaml
"{{ app_dir }}"
```

or:

```yaml
"{{ mysql_password }}"
```

This is **Jinja templating syntax**.

Basically:

```text
{{ variable_name }}
```

means:

> replace this with the variable's value.

For example:

```yaml
app_dir: /opt/cloud1
```

and later:

```yaml
path: "{{ app_dir }}"
```

becomes effectively:

```yaml
path: /opt/cloud1
```

Another example:

```yaml
mysql_database: wordpress
```

Then:

```yaml
MYSQL_DATABASE: "{{ mysql_database }}"
```

becomes:

```yaml
MYSQL_DATABASE: "wordpress"
```

---

# 23. What is `site.yml`?

Now we reach the main deployment file:

```text
site.yml
```

Think of it as the **entry point**.

When you execute:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Ansible starts reading this file.

Your structure is essentially:

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

Now every line makes sense.

---

# 24. `name:`

```yaml
name: Deploy Cloud-1 WordPress stack
```

This is mainly a human-readable description.

That's why terminal output says:

```text
PLAY [Deploy Cloud-1 WordPress stack]
```

It doesn't perform some special technical operation.

It's there so humans understand what the play does.

---

# 25. `hosts: webserver`

This:

```yaml
hosts: webserver
```

means:

> execute this play against the `[webserver]` group from `inventory.ini`.

So:

```text
site.yml
hosts: webserver
        |
        v
inventory.ini

[webserver]
cloud1-azure → 74.248.37.10
```

---

# 26. What does `become: true` mean?

You need root privileges on the **remote server** for operations like:

```text
apt install
editing /etc/...
UFW
Docker
/opt/cloud1
systemd
```

But you connect as:

```text
azureuser
```

not root.

Ansible's:

```yaml
become: true
```

means roughly:

> “Use privilege escalation on the remote server.”

Usually that means sudo.

Conceptually:

```text
Ansible SSH connection:
azureuser
    |
    | become: true
    v
sudo/root privileges
```

This is unrelated to you not having sudo on your **42 computer**.

There are two completely different systems:

```text
42 MACHINE                   AZURE SERVER
-----------                  ------------
You may have no sudo         azureuser has sudo
Ansible runs here      --->  become:true acts here
```

That distinction is crucial.

---

# 27. What is a playbook?

`site.yml` is a **playbook**.

A playbook tells Ansible:

```text
Which hosts?
What permissions?
What roles/tasks should happen?
In what order?
```

You run a playbook using:

```bash
ansible-playbook site.yml
```

Notice there are two commands:

```text
ansible
```

and:

```text
ansible-playbook
```

They are related but not identical.

For a quick single module:

```bash
ansible webserver -m ping
```

For a complete YAML playbook:

```bash
ansible-playbook site.yml
```

---

# 28. What is a role?

Your `site.yml` has:

```yaml
roles:
  - docker
  - firewall
  - tls
  - wordpress
```

A role is basically a logical module of infrastructure.

Instead of creating one huge:

```text
3000-line-site.yml
```

we have:

```text
docker
firewall
tls
wordpress
```

This also directly satisfies the evaluator's “modular architecture” requirement.

Conceptually:

```text
site.yml
  |
  +---- docker role
  |
  +---- firewall role
  |
  +---- tls role
  |
  +---- wordpress role
```

---

# 29. Why this particular order?

Your playbook executes:

```text
1. docker
2. firewall
3. tls
4. wordpress
```

There is reasoning behind that.

Docker first because later we need:

```bash
docker compose ...
```

Firewall next so external access is restricted.

TLS before Nginx because Nginx's configuration refers to certificate files:

```text
fullchain.pem
privkey.pem
```

Then WordPress role generates Compose/Nginx configuration and launches the containers.

So:

```text
Docker
  ↓
Firewall
  ↓
Certificates
  ↓
Containers / WordPress
```

---

# 30. What is a task?

Inside a role you see:

```yaml
- name: Install UFW
  ansible.builtin.apt:
    name: ufw
    state: present
```

That's an **Ansible task**.

Think:

```text
role
 └── sequence of tasks
```

For example firewall role:

```text
Install UFW
      ↓
Default deny incoming
      ↓
Allow outgoing
      ↓
Allow 22
Allow 80
Allow 443
      ↓
Enable UFW
```

That's exactly what your deployment log showed: Docker tasks succeeded, then firewall tasks succeeded, then TLS, then WordPress.

---

# 31. What is `ansible.builtin.apt`?

Example:

```yaml
ansible.builtin.apt:
```

This is an **Ansible module**.

Instead of doing:

```bash
sudo apt install ufw
```

we tell Ansible:

```yaml
ansible.builtin.apt:
  name: ufw
  state: present
```

This means:

> “Ensure package `ufw` exists.”

Notice the wording:

```text
Ensure
```

not:

```text
Install blindly every time
```

This takes us to one of the most important concepts in your evaluation.

---

# 32. Idempotency

You WILL want to understand this for defense.

Suppose we have:

```yaml
state: present
```

First run:

```text
UFW missing
     ↓
Ansible installs it
```

Second run:

```text
UFW already exists
     ↓
Nothing needs changing
```

Third run:

```text
still installed
     ↓
Nothing needs changing
```

The final state stays the same.

That's the idea of **idempotency**.

Mathematically, you can think:

```text
apply(configuration)
apply(configuration)
apply(configuration)
```

should lead to the same desired infrastructure state.

The subject explicitly requires that rerunning the installation produces the same working result.

That's why during evaluation we'll run:

```bash
ansible-playbook site.yml --ask-vault-pass
```

and then run it **again**.

---

# 33. What do `ok`, `changed`, `skipping`, and `failed` mean?

Your output has things like:

```text
ok: [cloud1-azure]
changed: [cloud1-azure]
skipping: [cloud1-azure]
failed: [cloud1-azure]
```

Understand these.

### `ok`

Example:

```text
ok: [cloud1-azure]
```

means:

> The desired state already exists / task succeeded without needing a modification.

On a second idempotent run, you want lots of `ok`.

---

### `changed`

```text
changed: [cloud1-azure]
```

means:

> The task succeeded and changed something on the remote machine.

For example first deployment:

```text
certificate didn't exist
→ created certificate
→ changed
```

Perfectly normal.

---

### `skipping`

```text
skipping: [cloud1-azure]
```

does **not** mean failure.

It means a `when:` condition told Ansible that task isn't applicable.

Your logs had:

```text
Install OpenSSL for self-signed certificates
skipping
```

Why?

Because your server uses:

```yaml
use_letsencrypt: true
```

So:

```text
use_letsencrypt = true

if NOT use_letsencrypt:
    generate self-signed certificate
```

False condition → skip.

That's correct.

---

### `failed`

```text
FAILED
```

means something genuinely went wrong.

Your WP-CLI problem ended with:

```text
failed=1
```

and the exact failure was:

```text
/usr/local/bin/docker-entrypoint.sh:
exec: line 11: core: not found
```



---

# 34. `register`

You'll see syntax like:

```yaml
register: mysql_ready
```

This means:

> “Save the result of the previous command into a variable named `mysql_ready`.”

Commands produce information such as:

```text
return code
stdout
stderr
changed
```

So later you can do:

```yaml
until: mysql_ready.rc == 0
```

`rc` means:

```text
return code
```

Unix convention:

```text
0     = success
non-0 = something failed / false
```

---

# 35. `retries`, `delay`, `until`

Your MySQL wait logic uses the equivalent of:

```yaml
retries: 30
delay: 2
until: mysql_ready.rc == 0
```

Why?

Starting a Docker container doesn't mean MySQL is immediately ready.

Timeline:

```text
docker starts container
      ↓
MySQL process launches
      ↓
MySQL initializes files
      ↓
MySQL initializes database
      ↓
MySQL accepts connections
```

That can take several seconds.

So instead of doing something bad like:

```text
sleep 60 seconds no matter what
```

we actually check:

```text
Is MySQL ready?
No → wait 2 sec
Is MySQL ready?
No → wait 2 sec
Is MySQL ready?
Yes → continue
```

Your first deployment showed exactly that:

```text
FAILED - RETRYING...
FAILED - RETRYING...
FAILED - RETRYING...
ok
```

That's **not deployment failure**. MySQL simply wasn't ready yet, and then it became ready.

---

# 36. `when`

You'll see:

```yaml
when: use_letsencrypt
```

This means:

```text
Execute this task only if use_letsencrypt is true.
```

And:

```yaml
when: not use_letsencrypt
```

means:

```text
execute only if it's false.
```

Another one:

```yaml
when: wordpress_installed.rc != 0
```

means:

> Only install WordPress if the check says WordPress isn't already installed.

That's another idempotency mechanism.

---

# 37. `changed_when: false`

Sometimes we execute a command just to inspect information.

For example:

```bash
dpkg --print-architecture
```

It doesn't modify anything.

But Ansible's generic command module can't always automatically know whether a command changed something.

So we explicitly say:

```yaml
changed_when: false
```

Meaning:

> “This operation is just a check. Never report it as changing the machine.”

This helps make our playbook accurately reflect idempotency.

---

# 38. `failed_when: false`

Your WordPress installation check does something conceptually like:

```bash
wp core is-installed
```

If WordPress isn't installed, WP-CLI returns non-zero.

Normally Ansible sees non-zero and says:

```text
FAILED
```

But in this case:

```text
WordPress isn't installed
```

isn't an unexpected deployment error.

It's information we want:

```text
Not installed?
      ↓
Install it.
```

So:

```yaml
failed_when: false
```

means:

> “Don't abort the playbook just because this check returned non-zero.”

---

# 39. What is Docker doing in this project?

Once Ansible prepares the server, Docker runs the actual application.

Your architecture is:

```text
                      INTERNET
                         |
                    80 / 443
                         |
                         v
                  +-------------+
                  |    NGINX    |
                  +-------------+
                   /           \
                  /             \
                 v               v
        +-------------+   +-------------+
        |  WORDPRESS  |   | PHPMYADMIN  |
        | PHP-FPM     |   |             |
        +-------------+   +-------------+
                 \             /
                  \           /
                   v         v
                  +-------------+
                  |    MYSQL    |
                  +-------------+
```

This matches the subject's requirement for separate processes/services in separate containers.

---

# 40. Why four permanent containers?

You have:

```text
cloud1_nginx
cloud1_wordpress
cloud1_phpmyadmin
cloud1_mysql
```

Each one has a separate job.

`nginx` handles web traffic.

`wordpress` runs PHP through PHP-FPM.

`phpmyadmin` provides a browser interface for MySQL.

`mysql` stores WordPress's database.

That's modular container architecture.

---

# 41. Why isn't WordPress itself listening on port 80?

You use:

```text
wordpress:php8.2-fpm
```

FPM is:

```text
FastCGI Process Manager
```

It processes PHP.

It is **not the public web server**.

It listens internally around:

```text
9000
```

Nginx receives:

```text
GET /index.php
```

and sends PHP execution to:

```text
wordpress:9000
```

So:

```text
browser
   ↓
nginx
   ↓
PHP request
   ↓
wordpress PHP-FPM
```

This cleanly separates Nginx and PHP.

---

# 42. Why does MySQL say `3306/tcp` but isn't publicly exposed?

Your earlier `docker ps` showed:

```text
cloud1_mysql
3306/tcp, 33060/tcp
```

This does **not** mean:

```text
Internet → 3306
```

Compare these two.

Nginx:

```text
0.0.0.0:80->80/tcp
```

means:

```text
HOST PORT 80
     ↓
CONTAINER PORT 80
```

So it is published.

MySQL:

```text
3306/tcp
```

with no:

```text
0.0.0.0:3306->3306
```

means the port exists **inside Docker**, but it isn't published from the host.

That is exactly what we want.

The subject explicitly says the database must not be directly accessible from the Internet.

---

# 43. Docker networking

We have conceptually two networks:

```text
frontend
backend
```

Nginx:

```text
frontend
```

WordPress:

```text
frontend + backend
```

phpMyAdmin:

```text
frontend + backend
```

MySQL:

```text
backend
```

Diagram:

```text
            FRONTEND
--------------------------------
     nginx
       |
       +----------+
       |          |
  wordpress   phpMyAdmin
       |          |
-------+----------+---------------
            BACKEND
       |          |
       +--- MySQL-+
```

The backend is marked:

```yaml
internal: true
```

which adds another boundary around the database-side network.

---

# 44. How does WordPress find MySQL?

This is a powerful Docker Compose concept.

WordPress gets:

```text
WORDPRESS_DB_HOST=mysql
```

Why `mysql`?

Because in Compose:

```yaml
services:
  mysql:
```

the service name:

```text
mysql
```

becomes a DNS hostname inside that Docker network.

So WordPress doesn't need:

```text
172.18.0.4
```

which could change.

It simply connects to:

```text
mysql:3306
```

Docker resolves `mysql` to the correct container.

---

# 45. What are Docker volumes?

Containers should be disposable.

If we delete:

```text
cloud1_mysql
```

we don't want to lose the entire site database.

So MySQL stores its database in:

```text
mysql_data
```

WordPress files live in:

```text
wordpress_data
```

Conceptually:

```text
MYSQL CONTAINER
     |
     v
mysql_data volume
     |
 survives container deletion/recreation
```

Similarly:

```text
WORDPRESS CONTAINER
     |
     v
wordpress_data
```

That is why:

```bash
docker compose down
docker compose up -d
```

does not automatically destroy the site.

But:

```bash
docker compose down -v
```

**does destroy volumes**.

That's why I told you to use `-v` only when you explicitly wanted a completely fresh installation.

The subject specifically requires data such as users, posts and images to survive reboot.

---

# 46. What does `restart: unless-stopped` do?

Each long-running service has:

```yaml
restart: unless-stopped
```

Meaning, broadly:

> If Docker restarts, start this container again unless the administrator deliberately stopped it.

And the Docker service itself is configured to start on server boot.

Therefore:

```text
Azure reboots
    ↓
Linux starts
    ↓
Docker service starts
    ↓
Docker restarts containers
    ↓
WordPress returns
```

Volumes still contain the data.

That's how we satisfy the reboot requirement.

---

# 47. What is Docker Compose?

Instead of manually executing:

```bash
docker run mysql ...
docker run wordpress ...
docker run nginx ...
docker run phpmyadmin ...
```

we define the complete stack in one:

```text
docker-compose.yml
```

Then:

```bash
docker compose up -d
```

means roughly:

> Build/start the desired application stack described by this file.

`up`:

```text
create/start services
```

`-d`:

```text
detached mode
```

Meaning don't keep all container output attached to your terminal.

---

# 48. Why is our file called `.j2` locally?

You have:

```text
docker-compose.yml.j2
```

not directly:

```text
docker-compose.yml
```

`.j2` means it's a **Jinja template**.

For example:

```yaml
MYSQL_DATABASE: "{{ mysql_database }}"
```

Before sending it to the server, Ansible replaces variables.

Suppose:

```yaml
mysql_database: wordpress
```

Then the rendered server file gets:

```yaml
MYSQL_DATABASE: "wordpress"
```

So:

```text
LOCAL
docker-compose.yml.j2
       |
       | Ansible renders variables
       v
REMOTE SERVER
/opt/cloud1/docker-compose.yml
```

Important distinction.

You edit:

```text
roles/wordpress/templates/docker-compose.yml.j2
```

locally.

Ansible generates:

```text
/opt/cloud1/docker-compose.yml
```

remotely.

---

# 49. What is `/opt/cloud1`?

That's the deployment directory **on Azure**.

Your local project:

```text
cloud-1-test/ansible/
```

is not the same as:

```text
/opt/cloud1
```

Remote result:

```text
Azure Server

/opt/cloud1/
├── docker-compose.yml
├── nginx.conf
└── certs/
    ├── fullchain.pem
    └── privkey.pem
```

So:

```text
LOCAL SOURCE OF TRUTH
roles/.../*.j2
         |
         | Ansible
         v
REMOTE GENERATED FILES
/opt/cloud1/...
```

Usually don't manually edit `/opt/cloud1/docker-compose.yml`.

Why?

Because your next Ansible run can overwrite it from the template.

Make permanent changes in the Ansible source instead.

---

# 50. What is Nginx doing?

Browser requests arrive:

```text
https://abel-hid.duckdns.org
```

DNS turns that into your Azure IP.

Then:

```text
Browser
   ↓
Internet
   ↓
Azure IP
   ↓ :443
Nginx container
```

Nginx examines the requested URL.

For:

```text
/
```

WordPress handles the site.

For:

```text
/phpmyadmin/
```

Nginx proxies the request to:

```text
phpmyadmin
```

For PHP files:

```text
.php
```

Nginx sends execution to:

```text
wordpress:9000
```

So Nginx is the single public entry point.

---

# 51. HTTP versus HTTPS

HTTP:

```text
port 80
```

HTTPS:

```text
port 443
```

HTTPS is HTTP protected with TLS.

Your configured domain uses Let's Encrypt.

So ideally:

```text
http://abel-hid.duckdns.org
       |
       | 301 redirect
       v
https://abel-hid.duckdns.org
```

And HTTPS uses:

```text
certificate
private key
```

Nginx loads:

```text
fullchain.pem
privkey.pem
```

---

# 52. What is Let's Encrypt?

Let's Encrypt gives us a TLS certificate trusted by browsers.

The certificate essentially helps establish:

```text
“This server controls abel-hid.duckdns.org”
```

and enables encrypted HTTPS communication.

We use:

```text
certbot
```

to request the certificate.

Certbot needs to prove control of the domain.

That's why DNS must point:

```text
abel-hid.duckdns.org
        ↓
74.248.37.10
```

---

# 53. What is WP-CLI?

This is the part where our bug happened.

Normally when WordPress first launches, you might visit it in a browser and see:

```text
Choose language
Site title
Admin username
Admin password
Install WordPress
```

But that would mean part of deployment is manual.

Our goal is automation.

So we use:

```text
WP-CLI
```

which is WordPress's command-line interface.

Instead of clicking:

```text
Install WordPress
```

we execute conceptually:

```bash
wp core install \
    --url=... \
    --title=... \
    --admin_user=... \
    --admin_password=...
```

So WordPress is ready automatically.

---

# 54. What was our WP-CLI bug?

Our generated command was effectively:

```bash
docker compose --profile tools run --rm wpcli core install ...
```

Docker started the `wpcli` container and passed:

```text
core
```

as the executable.

The container replied:

```text
core: not found
```

Your log showed exactly that.

The intended executable is:

```text
wp
```

so we corrected it to:

```bash
docker compose --profile tools run --rm wpcli wp core install ...
```

Think:

```text
wp          ← program
core        ← command group
install     ← action
```

Similar idea:

```text
git branch create
```

where `git` must be the executable.

---

# 55. What is the `wpcli` container profile?

There are four things that should continuously run:

```text
nginx
wordpress
phpmyadmin
mysql
```

But WP-CLI isn't a server.

It is just a tool we need briefly.

So it doesn't need to sit running forever.

We put it under:

```yaml
profiles:
  - tools
```

Normal:

```bash
docker compose up -d
```

doesn't start it permanently.

When needed:

```bash
docker compose --profile tools run --rm wpcli ...
```

means:

```text
enable tools profile
      ↓
create temporary WP-CLI container
      ↓
run command
      ↓
--rm
      ↓
delete temporary container
```

So after installation you still expect only four long-running containers.

---

# 56. What does `--rm` mean there?

Here:

```bash
docker compose run --rm wpcli ...
```

`--rm` means:

> delete this temporary container after the command finishes.

It does **not** delete WordPress data because WordPress data is in the named volume.

---

# 57. What is a firewall?

Your Azure server is a machine on the Internet.

Potentially it has many services.

We want the public world to access only:

```text
22 SSH
80 HTTP
443 HTTPS
```

The subject explicitly requires those three and says other ports should be blocked.

Your UFW configuration conceptually is:

```text
Default incoming: DENY

Exception:
22   ALLOW
80   ALLOW
443  ALLOW
```

Then:

```text
3306 MySQL → not public
9000 PHP-FPM → not public
```

---

# 58. Why allow port 22 before enabling UFW?

This is a good evaluation question.

We're configuring the server through SSH.

If we did:

```text
deny everything
enable firewall
```

before permitting SSH, we could lock ourselves out.

So:

```text
allow 22
allow 80
allow 443
then enable firewall
```

Port 22 keeps Ansible's SSH connection possible.

---

# 59. What are ports conceptually?

One server IP can run many services.

Ports distinguish them.

Imagine an apartment building:

```text
IP address = building address
port       = apartment number
```

For your server:

```text
74.248.37.10:22
               ↓
              SSH

74.248.37.10:80
               ↓
              HTTP

74.248.37.10:443
               ↓
              HTTPS
```

If MySQL were public:

```text
74.248.37.10:3306
```

but we deliberately don't allow that.

---

# 60. Now your exact command sequence makes sense

Start from:

```text
cloud-1-test/
```

Enter your Ansible project:

```bash
cd ansible
```

You should now see:

```bash
pwd
```

something ending with:

```text
/cloud-1-test/ansible
```

Check:

```bash
ls
```

You want:

```text
ansible.cfg
group_vars
inventory.ini
roles
site.yml
```

Now test Ansible connectivity:

```bash
ansible webserver -m ping --ask-vault-pass
```

Read it as:

```text
ansible
    ↓
against webserver group
    ↓
execute ping module
    ↓
ask me for Vault decryption password
```

Expected:

```text
cloud1-azure | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Then deploy:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Read that as:

```text
ansible-playbook
        ↓
load site.yml
        ↓
load inventory
        ↓
load variables
        ↓
decrypt Vault
        ↓
SSH to cloud1-azure
        ↓
become root remotely
        ↓
docker role
        ↓
firewall role
        ↓
tls role
        ↓
wordpress role
```

---

# 61. How to read the full deployment

When you see:

```text
PLAY [Deploy Cloud-1 WordPress stack]
```

Ansible started the play.

Then:

```text
TASK [Gathering Facts]
```

Ansible asks the remote server things like:

```text
Which Linux distribution?
Which release?
Which architecture?
What network facts exist?
```

Those values become variables such as:

```text
ansible_distribution_release
```

Then:

```text
TASK [docker : Update apt cache]
```

means:

```text
role name: docker
task name: Update apt cache
```

Same:

```text
TASK [firewall : Install UFW]
```

means:

```text
role = firewall
task = Install UFW
```

Then TLS, then WordPress.

---

# 62. Understand the recap

At the bottom:

```text
PLAY RECAP
cloud1-azure : ok=30 changed=... unreachable=0 failed=0 ...
```

The most critical fields are:

```text
unreachable=0
failed=0
```

`unreachable > 0` usually means Ansible couldn't communicate with the server.

Examples:

```text
bad IP
bad SSH key
wrong username
server offline
port 22 blocked
```

`failed > 0` means Ansible reached the machine but some deployment task failed.

For a successful deployment:

```text
unreachable=0
failed=0
```

---

# 63. Why rerun the playbook?

After successful deployment:

```bash
ansible-playbook site.yml --ask-vault-pass
```

we run the **same exact command again**:

```bash
ansible-playbook site.yml --ask-vault-pass
```

We aren't doing that because we forgot we already ran it.

We are testing:

```text
IDEMPOTENCY
```

First run might say:

```text
changed=10
```

because infrastructure was created.

Second run should mostly say:

```text
ok
```

and fewer/no unnecessary changes.

Most importantly:

```text
failed=0
```

and the site remains identical.

This is directly part of your evaluation.

---

# 64. The entire system in one diagram

This is the mental picture I want you to have:

```text
                    YOUR 42 MACHINE
┌───────────────────────────────────────────────────────┐
│                                                       │
│ cloud-1-test/ansible/                                 │
│                                                       │
│ ansible.cfg                                           │
│      │                                                │
│      └── tells Ansible where inventory is             │
│                                                       │
│ inventory.ini                                         │
│      │                                                │
│      └── server IP / SSH user / SSH key / domain      │
│                                                       │
│ group_vars/                                           │
│      │                                                │
│      ├── normal variables                             │
│      └── encrypted passwords (Vault)                  │
│                                                       │
│ site.yml                                              │
│      │                                                │
│      ├── docker role                                  │
│      ├── firewall role                                │
│      ├── tls role                                     │
│      └── wordpress role                               │
│                                                       │
└──────────────────────────┬────────────────────────────┘
                           │
                           │ SSH :22
                           │
                           ▼
                    AZURE UBUNTU SERVER
┌───────────────────────────────────────────────────────┐
│                                                       │
│ Docker                                                │
│                                                       │
│                    Internet                           │
│                        │                              │
│                    80 / 443                           │
│                        │                              │
│                        ▼                              │
│                  ┌───────────┐                        │
│                  │   NGINX   │                        │
│                  └─────┬─────┘                        │
│                       / \                             │
│                      /   \                            │
│                     ▼     ▼                           │
│             ┌──────────┐ ┌──────────┐                 │
│             │WordPress │ │phpMyAdmin│                 │
│             │ PHP-FPM  │ │          │                 │
│             └────┬─────┘ └────┬─────┘                 │
│                  │            │                       │
│                  └─────┬──────┘                       │
│                        ▼                              │
│                  ┌───────────┐                        │
│                  │   MySQL   │                        │
│                  └───────────┘                        │
│                                                       │
│ Persistent volumes:                                  │
│   mysql_data                                          │
│   wordpress_data                                      │
│                                                       │
│ Public ports:                                         │
│   22  SSH                                             │
│   80  HTTP                                            │
│   443 HTTPS                                           │
│                                                       │
│ NOT public:                                           │
│   3306 MySQL                                          │
│   9000 PHP-FPM                                        │
│                                                       │
└───────────────────────────────────────────────────────┘
```

That diagram is basically your whole Cloud-1 project.

---

# 65. What you should know before the evaluator

Don't memorize YAML blindly. Be able to answer these concepts in your own words:

> **What is Ansible?**
> A configuration-management/automation tool. We run it from our local machine; it connects to remote Ubuntu servers over SSH and provisions them automatically.

> **What is inventory?**
> The file describing which remote hosts Ansible manages and how to connect to them.

> **What is a playbook?**
> A YAML definition describing what should be deployed to which hosts.

> **What is a role?**
> A modular group of related Ansible tasks, such as Docker, firewall, TLS, or WordPress.

> **What is idempotency?**
> Reapplying the same deployment should result in the same correct final state instead of breaking or duplicating configuration.

> **What is Ansible Vault?**
> Encryption for sensitive Ansible variables such as passwords, so plaintext secrets don't live in the repository.

> **What does `become: true` do?**
> It elevates privileges on the remote server, normally through sudo, for operations requiring root.

> **Why Docker Compose?**
> It declares and orchestrates the complete multi-container application in one configuration.

> **Why separate containers?**
> Nginx, WordPress/PHP-FPM, phpMyAdmin and MySQL each have separate responsibilities, matching the project's one-process/service-per-container requirement.

> **Why volumes?**
> They make WordPress and database data persist independently of container recreation and reboot.

> **Why is MySQL not exposed?**
> WordPress and phpMyAdmin communicate with MySQL through Docker's private network; there is no reason for the Internet to reach port 3306.

> **Why Nginx?**
> It is the public HTTP/HTTPS entry point, routes WordPress/PHP traffic and reverse-proxies `/phpmyadmin/`.

> **Why ports 22/80/443 only?**
> 22 is SSH for administration/Ansible, 80 is HTTP, and 443 is HTTPS. The subject explicitly restricts external access to those ports.

Now don't rush into ten testing commands. Your **next single step** should be:

```bash
cd cloud-1-test/ansible
```

Then verify:

```bash
pwd
ls
```

Then:

```bash
ansible webserver -m ping --ask-vault-pass
```

If you send me its output, I'll continue from there and explain **each thing we do and why we're doing it**, not just throw commands at you.
