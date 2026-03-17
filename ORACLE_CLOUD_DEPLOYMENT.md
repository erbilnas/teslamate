# TeslaMate on Oracle Cloud Free Tier – Public Access (Phone/Web)

This guide walks you through deploying TeslaMate on Oracle Cloud Free Tier with HTTPS and public access from your phone or any device.

## Upgrading from the Simple Docker Setup

If you previously ran the basic `docker-compose.yml` (without Traefik), use the **same** `TM_DB_PASS` and `GRAFANA_PW` in `.env` as your old database and Grafana passwords. Otherwise the database will be inaccessible.

---

## Prerequisites

- Oracle Cloud account (free tier at [cloud.oracle.com](https://cloud.oracle.com))
- A domain name pointing to your server (e.g. `teslamate.yourdomain.com`)
- Credit card for verification (not charged for free tier)

---

## Step 1: Create an Oracle Cloud VM

1. Sign in to [Oracle Cloud Console](https://cloud.oracle.com)
2. Create a **Compute Instance**:
   - **Shape**: Ampere (ARM) – choose "Flex" and set **2 OCPUs**, **12 GB RAM**
   - **Image**: Ubuntu 22.04 or 24.04
   - **Add SSH key**: Upload your public key or generate a new one
3. After creation, note the **public IP address**
4. In **Networking > Virtual Cloud Networks > your VCN > Security Lists**, add **Ingress** rules:
   - **Source**: 0.0.0.0/0, **Port**: 80 (HTTP)
   - **Source**: 0.0.0.0/0, **Port**: 443 (HTTPS)

---

## Step 2: Point Your Domain to the Server

Create an **A record** for your subdomain (e.g. `teslamate.yourdomain.com`) pointing to the VM’s public IP.

---

## Step 3: Install Docker on the VM

SSH into your VM:

```bash
ssh ubuntu@YOUR_PUBLIC_IP
```

Install Docker and Docker Compose:

```bash
# Update and install Docker
sudo apt update && sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to docker group
sudo usermod -aG docker $USER
# Log out and back in for group to take effect
```

---

## Step 4: Deploy TeslaMate

1. Copy this project to the VM (e.g. via `scp` or `git clone`):

```bash
# From your local machine
scp -r teslamate/ ubuntu@YOUR_PUBLIC_IP:~/
```

2. SSH into the VM and edit `.env`:

```bash
cd ~/teslamate
nano .env
```

Set these values:

| Variable | Value |
|----------|-------|
| `TM_ENCRYPTION_KEY` | Run `openssl rand -base64 32` and paste the result |
| `TM_DB_PASS` | Strong database password |
| `GRAFANA_PW` | Grafana admin password |
| `FQDN_TM` | Your domain (e.g. `teslamate.yourdomain.com`) |
| `LETSENCRYPT_EMAIL` | Your email for Let's Encrypt |

3. Create `.htpasswd` (not in repo for security):

```bash
# Option A: Copy from example (default password: changeme)
cp .htpasswd.example .htpasswd

# Option B: Generate with your own password
htpasswd -nbB teslamate YOUR_PASSWORD > .htpasswd
```

Change the default `changeme` before going live.

4. Start the stack:

```bash
docker compose up -d
```

5. Check logs for certificate issuance:

```bash
docker compose logs -f proxy
```

---

## Step 5: Configure TeslaMate

1. Open `https://teslamate.yourdomain.com` in a browser (or on your phone)
2. Log in with the Basic Auth credentials (default: `teslamate` / `changeme`)
3. In **Settings**, set:
   - **Web App URL**: `https://teslamate.yourdomain.com`
   - **Grafana URL**: `https://teslamate.yourdomain.com/grafana`
4. Sign in with your Tesla account
5. Grafana dashboards: `https://teslamate.yourdomain.com/grafana` (use `GRAFANA_USER` / `GRAFANA_PW` from `.env`)

---

## Access from Phone

- **TeslaMate**: `https://teslamate.yourdomain.com` (Basic Auth, then Tesla login)
- **Grafana**: `https://teslamate.yourdomain.com/grafana`

Add these to your phone’s home screen or bookmarks for quick access.

---

## Troubleshooting

### Let's Encrypt certificate fails

- Ensure ports 80 and 443 are open in the Oracle Cloud Security List
- Confirm your domain A record points to the VM’s public IP
- Check: `docker compose logs proxy`

### Cannot connect to Tesla API

- Ensure the VM has outbound internet access
- Oracle Free Tier allows 10 TB egress/month

### Reset Grafana admin password

```bash
docker compose exec grafana grafana-cli admin reset-admin-password NEW_PASSWORD
```

---

## Security Checklist

- [ ] Replace `TM_ENCRYPTION_KEY` with a strong random key
- [ ] Use strong `TM_DB_PASS` and `GRAFANA_PW`
- [ ] Change `.htpasswd` password from default `changeme`
- [ ] Keep `.env` and `.htpasswd` out of version control (they are in `.gitignore`)
