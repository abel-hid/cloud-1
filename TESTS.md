
# Check Python first
python3 --version

# Create a local virtual environment
python3 -m venv ~/.ansible-venv

# Activate it
source ~/.ansible-venv/bin/activate

# Upgrade pip
python3 -m pip install --upgrade pip

# Install old cryptography
python3 -m pip install --only-binary=:all: "cryptography==42.0.8"

# Install old Ansible
python3 -m pip install "ansible==8.7.0"

# Check that it works
ansible --version
ansible-playbook --version

-------------------



Yes. Use this order so we test the project almost exactly as an evaluator would.

The subject requires the deployment to work from the automation, survive reboot with persistent data, keep services in separate containers, expose only 22/80/443, be idempotent, and avoid hard-coded secrets.

## 1. Extract the project

On your computer:

```bash
unzip cloud-1-completed.zip
cd cloud-1/ansible
```

You should have:

```bash
ls
```

Expected:

```text
ansible.cfg
group_vars
inventory.ini
roles
site.yml
```

Your current configured server is:

```text
IP:     74.248.37.10
Domain: abel-hid.duckdns.org
SSH:    azureuser
Key:    ~/.ssh/id_rsa
```

---

## 2. First test SSH before Ansible

Run:

```bash
ssh -i ~/.ssh/id_rsa azureuser@74.248.37.10
```

If it connects:

```bash
exit
```

Then test with Ansible:

```bash
ansible webserver -m ping
```

Expected:

```text
cloud1-azure | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If your SSH key is stored somewhere else, edit:

```bash
nano inventory.ini
```

and change:

```ini
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

to the correct path.

---

# 3. Run the complete deployment

Your Vault password file should stay **outside the repository**.

For example, if you downloaded it into `~/Downloads`:

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Downloads/cloud-1-vault-password.txt
```

Or interactively:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Then enter the password from:

```text
cloud-1-vault-password.txt
```

The play should finish with something similar to:

```text
PLAY RECAP
cloud1-azure : ok=... changed=... unreachable=0 failed=0 skipped=...
```

The important part is:

```text
unreachable=0
failed=0
```

---

# 4. Immediately run it AGAIN — idempotency test

This is important for evaluation.

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Downloads/cloud-1-vault-password.txt
```

It must finish successfully again:

```text
failed=0
```

The site should remain exactly the same.

That directly tests the subject requirement that running the installation multiple times produces the same working result.

---

# 5. Open the website in browser

Open:

```text
https://abel-hid.duckdns.org
```

You should see the WordPress site.

Also test HTTP:

```text
http://abel-hid.duckdns.org
```

Because this host uses Let's Encrypt, HTTP should redirect to:

```text
https://abel-hid.duckdns.org
```

Refresh the page several times.

The website should continue working.

---

# 6. WordPress admin in browser

Open:

```text
https://abel-hid.duckdns.org/wp-admin/
```

Username:

```text
cloud1_admin
```

The password is stored inside Ansible Vault.

To see it:

```bash
ansible-vault view group_vars/all/vault.yml \
  --vault-password-file ~/Downloads/cloud-1-vault-password.txt
```

You'll see variables including:

```yaml
vault_wordpress_admin_password: ...
vault_mysql_root_password: ...
vault_mysql_password: ...
```

Use the value of:

```text
vault_wordpress_admin_password
```

for WordPress.

### Browser test

Log in.

Then create something simple:

```text
Posts
→ Add New
→ Title: Cloud-1 test
→ Publish
```

Go back to:

```text
https://abel-hid.duckdns.org
```

Confirm the content exists.

This post will also be useful later for the persistence/reboot test.

---

# 7. Test phpMyAdmin

Open:

```text
https://abel-hid.duckdns.org/phpmyadmin/
```

Login with:

```text
Username: wp_user
Password: vault_mysql_password
```

Get the password again with:

```bash
ansible-vault view group_vars/all/vault.yml \
  --vault-password-file ~/Downloads/cloud-1-vault-password.txt
```

Inside phpMyAdmin you should see:

```text
wordpress
```

database.

You should **not** need:

```text
74.248.37.10:8080
```

or anything similar.

phpMyAdmin is deliberately accessed through Nginx at:

```text
/phpmyadmin/
```

---

# 8. Test HTTP → HTTPS from terminal

From your computer:

```bash
curl -I http://abel-hid.duckdns.org
```

Expected approximately:

```text
HTTP/1.1 301 Moved Permanently
Location: https://abel-hid.duckdns.org/...
```

Now:

```bash
curl -I https://abel-hid.duckdns.org
```

Expected:

```text
HTTP/2 200
```

or:

```text
HTTP/1.1 200 OK
```

Test phpMyAdmin:

```bash
curl -I https://abel-hid.duckdns.org/phpmyadmin/
```

You should get a valid HTTP response, normally `200`.

---

# 9. SSH into server for container tests

```bash
ssh -i ~/.ssh/id_rsa azureuser@74.248.37.10
```

Then:

```bash
cd /opt/cloud1
```

---

# 10. See all containers

Run:

```bash
sudo docker compose ps
```

We expect these four permanent containers:

```text
cloud1_mysql
cloud1_wordpress
cloud1_phpmyadmin
cloud1_nginx
```

All should be:

```text
Up
```

You can also run:

```bash
sudo docker ps
```

The architecture is intentionally:

```text
Browser
   |
   v
 nginx
 /    \
v      v
wordpress phpmyadmin
   \      /
      mysql
```

`wpcli` should **not** be permanently running. It is only a setup tool.

This matches the subject's one-process/service-per-container approach.

---

# 11. Test all containers start in one command

From:

```bash
cd /opt/cloud1
```

Run:

```bash
sudo docker compose down
```

Then:

```bash
sudo docker compose up -d
```

Check:

```bash
sudo docker compose ps
```

Then browser:

```text
https://abel-hid.duckdns.org
```

Your WordPress post must still exist.

### Important

`docker compose down` does **not** delete our named volumes unless you add `-v`.

So DO NOT run:

```bash
sudo docker compose down -v
```

That would intentionally delete persistent volumes.

---

# 12. Test individual container restart

The evaluator asks whether services can be restarted individually.

For example:

```bash
sudo docker compose restart wordpress
```

Then:

```bash
sudo docker compose ps
```

Test Nginx:

```bash
sudo docker compose restart nginx
```

Test MySQL:

```bash
sudo docker compose restart mysql
```

Test phpMyAdmin:

```bash
sudo docker compose restart phpmyadmin
```

Then:

```bash
sudo docker compose ps
```

Everything should return to:

```text
Up
```

And browser should still work:

```text
https://abel-hid.duckdns.org
```

---

# 13. Inspect the volumes

Run:

```bash
sudo docker volume ls
```

You should see volumes corresponding to:

```text
mysql_data
wordpress_data
```

Usually Compose will prefix them, for example:

```text
cloud1_mysql_data
cloud1_wordpress_data
```

You can get the exact names with:

```bash
sudo docker compose config --volumes
```

Expected:

```text
mysql_data
wordpress_data
```

Inspect them:

```bash
sudo docker volume inspect cloud1_mysql_data
```

and:

```bash
sudo docker volume inspect cloud1_wordpress_data
```

The exact generated names can be confirmed first using:

```bash
sudo docker volume ls
```

This proves that database data and WordPress files are stored outside container writable layers.

---

# 14. Test persistence properly

Before rebooting, make sure the WordPress test post exists.

Browser:

```text
https://abel-hid.duckdns.org
```

Now on server:

```bash
sudo reboot
```

Your SSH session will disconnect.

Wait around 30–60 seconds.

Then from your computer:

```bash
ssh -i ~/.ssh/id_rsa azureuser@74.248.37.10
```

Once connected:

```bash
sudo docker ps
```

The four containers should automatically be running again because Docker itself is enabled and each service has:

```yaml
restart: unless-stopped
```

Now go again to:

```text
https://abel-hid.duckdns.org
```

The test post must still exist.

Also:

```text
https://abel-hid.duckdns.org/wp-admin/
```

Your user account must still exist.

This tests both mandatory conditions: automatic restart and preservation of site data after reboot.

---

# 15. Test firewall

On server:

```bash
sudo ufw status verbose
```

You should see only:

```text
22/tcp   ALLOW
80/tcp   ALLOW
443/tcp  ALLOW
```

The default incoming policy should be:

```text
deny
```

You can also run:

```bash
sudo ss -lntp
```

The externally relevant ports should be:

```text
22
80
443
```

---

# 16. Very important: prove MySQL is NOT exposed

From **your own computer**, not from inside the server:

```bash
nc -vz 74.248.37.10 3306
```

It should **fail**.

Something such as:

```text
Connection refused
```

or:

```text
Operation timed out
```

is good.

Now test the public ports:

```bash
nc -vz 74.248.37.10 22
nc -vz 74.248.37.10 80
nc -vz 74.248.37.10 443
```

Those should succeed.

So the expected result is roughly:

```text
22    OPEN
80    OPEN
443   OPEN
3306  CLOSED/FILTERED
```

The subject explicitly requires only 22, 80 and 443 to be externally accessible.

---

# 17. Prove Docker itself isn't publishing MySQL

On server:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Expected idea:

```text
NAMES                  PORTS
cloud1_nginx            0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
cloud1_wordpress        9000/tcp
cloud1_phpmyadmin       80/tcp
cloud1_mysql            3306/tcp
```

The important distinction:

```text
cloud1_nginx:
0.0.0.0:80
0.0.0.0:443
```

but MySQL should **NOT** show:

```text
0.0.0.0:3306
```

`3306/tcp` by itself means Docker knows the container port, but it is **not published to the host**.

---

# 18. Check Docker networks

Run:

```bash
sudo docker network ls
```

Then:

```bash
sudo docker network inspect cloud1_backend
```

and:

```bash
sudo docker network inspect cloud1_frontend
```

If you want to get exact generated names first:

```bash
sudo docker network ls | grep cloud1
```

Backend should report:

```json
"Internal": true
```

MySQL is attached only to the backend network.

---

# 19. Test MySQL from WordPress network

Although MySQL must not be public, the containers need to reach it.

Run:

```bash
cd /opt/cloud1
sudo docker compose exec wordpress getent hosts mysql
```

It should resolve an internal Docker IP.

Likewise:

```bash
sudo docker compose exec phpmyadmin getent hosts mysql
```

That proves inter-container DNS/networking is working.

---

# 20. Check container logs

Run:

```bash
sudo docker compose logs --tail=50
```

Or individually:

```bash
sudo docker compose logs --tail=50 nginx
```

```bash
sudo docker compose logs --tail=50 wordpress
```

```bash
sudo docker compose logs --tail=50 mysql
```

```bash
sudo docker compose logs --tail=50 phpmyadmin
```

You don't want repeating fatal errors or containers constantly restarting.

Check status:

```bash
sudo docker compose ps
```

---

# 21. Test Nginx configuration

Run:

```bash
sudo docker compose exec nginx nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

---

# 22. Test database directly

You can test MySQL without exposing its password in your shell command:

```bash
sudo docker compose exec mysql sh -c \
'mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"'
```

Expected:

```text
mysqld is alive
```

Check the WordPress database:

```bash
sudo docker compose exec mysql sh -c \
'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW DATABASES;"'
```

You should see:

```text
wordpress
```

---

# 23. Test WordPress installation with WP-CLI

Run:

```bash
sudo docker compose --profile tools run --rm wpcli \
  core is-installed --path=/var/www/html
```

Then:

```bash
echo $?
```

Expected:

```text
0
```

You can also get the site URL:

```bash
sudo docker compose --profile tools run --rm wpcli \
  option get siteurl --path=/var/www/html
```

Expected:

```text
https://abel-hid.duckdns.org
```

---

# 24. Test TLS certificate

From your computer:

```bash
openssl s_client \
  -connect abel-hid.duckdns.org:443 \
  -servername abel-hid.duckdns.org </dev/null
```

Or easier:

```bash
curl -Iv https://abel-hid.duckdns.org
```

You want successful TLS negotiation without a certificate warning/error.

In the browser, click the lock icon and check that HTTPS is valid.

---

# 25. Check secret management

Back on your project machine:

```bash
cd cloud-1/ansible
cat group_vars/all/vault.yml
```

It should look encrypted, beginning approximately with:

```text
$ANSIBLE_VAULT;1.1;AES256
```

It should **not** show plaintext database passwords.

To intentionally decrypt it:

```bash
ansible-vault view group_vars/all/vault.yml \
  --vault-password-file ~/Downloads/cloud-1-vault-password.txt
```

The subject specifically forbids hard-coded secrets.

---

# 26. The short evaluator sequence

When you're doing the actual defense, these are the commands I would keep ready.

On your computer:

```bash
cd cloud-1/ansible
```

```bash
ansible webserver -m ping
```

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Downloads/cloud-1-vault-password.txt
```

Run it **again**:

```bash
ansible-playbook site.yml \
  --vault-password-file ~/Downloads/cloud-1-vault-password.txt
```

Then:

```bash
curl -I http://abel-hid.duckdns.org
```

```bash
curl -I https://abel-hid.duckdns.org
```

```bash
nc -vz 74.248.37.10 3306
```

SSH:

```bash
ssh -i ~/.ssh/id_rsa azureuser@74.248.37.10
```

Then on server:

```bash
cd /opt/cloud1
```

```bash
sudo docker compose ps
```

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

```bash
sudo docker volume ls
```

```bash
sudo docker network ls
```

```bash
sudo ufw status verbose
```

```bash
sudo docker compose exec nginx nginx -t
```

```bash
sudo docker compose exec mysql sh -c \
'mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"'
```

Then browser:

```text
https://abel-hid.duckdns.org
https://abel-hid.duckdns.org/wp-admin/
https://abel-hid.duckdns.org/phpmyadmin/
```

Finally test reboot:

```bash
sudo reboot
```

Reconnect:

```bash
ssh -i ~/.ssh/id_rsa azureuser@74.248.37.10
```

Check:

```bash
sudo docker ps
```

and refresh the browser. **Your WordPress post must still be there.**

That gives us a clean test of essentially every mandatory evaluation point without adding unrelated functionality.




---

