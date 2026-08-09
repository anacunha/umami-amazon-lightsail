# umami-amazon-lightsail

Self-hosted [Umami](https://umami.is) analytics on an AWS Lightsail instance, with [Caddy](https://caddyserver.com) handling HTTPS automatically. Umami's official `docker-compose.yml` lives here as an untouched git submodule (`umami/`). `docker-compose.override.yml` adds Caddy and secrets on top of it.

## Prerequisites

1. Create a Lightsail instance (Platform: Linux, Blueprint: Ubuntu)
2. Attach a static IP: **Networking** tab → **IP addresses** → **Attach static IP**
3. Firewall: new instances open with only 80 and 22 by default. Add a rule for port 443 (it's not there yet), and restrict port 22 to Lightsail browser SSH only
4. Create a DNS record pointing your domain at the static IP:
   - Type: **A**
   - Name/Host: your subdomain (e.g. `analytics`)
   - Value: the static IP from step 2
   - TTL: 300 (or your provider's default)

Then connect to the instance via SSH and continue below.

### Add backup memory (optional)

If Postgres, Umami, and Caddy all need memory at the same time and the instance runs out of RAM, Linux starts killing processes to free it up. This can silently take your site down. A swap file gives it disk space to fall back on instead, so things slow down rather than crash:

```
# reserve 1GB of disk space for the swap file
sudo fallocate -l 1G /swapfile
# root-only access to the swap file
sudo chmod 600 /swapfile
# format the file as swap space
sudo mkswap /swapfile
# activate it now
sudo swapon /swapfile
# re-activate automatically on every reboot
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Install Docker

```
# refresh the package list
sudo apt update
# install the Docker engine and compose plugin
sudo apt install -y docker.io docker-compose-v2
# start Docker now, and on every future boot
sudo systemctl enable --now docker
# run docker without sudo
sudo usermod -aG docker $USER
# apply the group change without logging out
newgrp docker
```

## Deploy

```
git clone --recurse-submodules https://github.com/anacunha/umami-amazon-lightsail.git ~/umami-deploy
cd ~/umami-deploy
# docker compose only auto-merges override files in the same folder
cp docker-compose.override.yml Caddyfile umami/
```

First deploy only: generate secrets and set your domain:

```
cd umami
echo "APP_SECRET=$(openssl rand -hex 32)" > .env
echo "POSTGRES_PASSWORD=$(openssl rand -hex 16)" >> .env
```

**Replace `analytics.yourdomain.com` with your actual domain before running this:**

```
echo "DOMAIN=analytics.yourdomain.com" >> .env
```

Bring it up:

```
# download the images
docker compose pull
# create and start all three containers, in the background
docker compose up -d
# stream logs live to watch startup
docker compose logs -f
```

Open `https://analytics.yourdomain.com` and log in with `admin` / `umami`.

> ⚠️ **Change the password now.** The default login is the same on every Umami install.

- Change your password: https://docs.umami.is/docs/login
- Add a website to track: https://docs.umami.is/docs/add-a-website
- Add the tracking script to your site: https://docs.umami.is/docs/collect-data

## Updating

On the instance:

```
cd ~/umami-deploy
git pull --recurse-submodules
cp docker-compose.override.yml Caddyfile umami/
cd umami
docker compose pull
docker compose up -d --force-recreate
```

Your data isn't affected by this update.

If you forked this repo, Dependabot opens a pull request on your fork whenever Umami has new commits. Merge it, then run the steps above to deploy the update.
