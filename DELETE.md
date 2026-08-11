Run this **on the server**. It deletes all Cloud-1 deployment state so the next Ansible run starts fresh, while leaving Docker itself installed:

```bash
sudo sh -c '
if [ -f /opt/cloud1/docker-compose.yml ]; then
  cd /opt/cloud1 && docker compose down -v --remove-orphans || true
fi

docker ps -aq --filter "name=cloud1_" | xargs -r docker rm -f
docker volume ls -q | grep "^cloud1_" | xargs -r docker volume rm
docker network ls --format "{{.Name}}" | grep "^cloud1_" | xargs -r docker network rm

rm -rf /opt/cloud1

rm -rf /etc/letsencrypt/live/abel-hid.duckdns.org
rm -rf /etc/letsencrypt/archive/abel-hid.duckdns.org
rm -f /etc/letsencrypt/renewal/abel-hid.duckdns.org.conf

echo "Cloud-1 completely removed."
'
```

This removes the Cloud-1 containers, **MySQL/WordPress persistent data**, Cloud-1 networks, `/opt/cloud1`, and the project's TLS certificate. It does **not** uninstall Docker or delete unrelated containers/images.

Afterward, verify with:

```bash
sudo docker ps -a | grep cloud1 || echo "No containers"
sudo docker volume ls | grep cloud1 || echo "No volumes"
sudo docker network ls | grep cloud1 || echo "No networks"
test -e /opt/cloud1 && echo "Directory exists" || echo "No /opt/cloud1"
```

Then you can redeploy from your local machine with:

```bash
ansible-playbook site.yml --ask-vault-pass
```

That gives you a genuinely fresh Cloud-1 installation.
