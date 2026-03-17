# TeslaMate on Hetzner Cloud – Public Access (Phone/Web)

This guide walks you through deploying TeslaMate on Hetzner Cloud with HTTPS and public access from your phone or any device.

## Upgrading from the Simple Docker Setup

If you previously ran the basic `docker-compose.yml` (without Traefik), use the **same** `TM_DB_PASS` and `GRAFANA_PW` in `.env` as your old database and Grafana passwords. Otherwise the database will be inaccessible.

---

## Prerequisites

- Hetzner Cloud account ([console.hetzner.cloud](https://console.hetzner.cloud))
- A domain name pointing to your server (e.g. `teslamate.yourdomain.com`)
- Payment method (Hetzner is pay-as-you-go, ~€14+/month for a suitable server)

---

## Step 1: Create a Hetzner Cloud Server

1. Sign in to [Hetzner Cloud Console](https://console.hetzner.cloud)
2. Create a **Server**:
   - **Location**: Choose a region (e.g. Falkenstein, Nuremberg, Helsinki)
   - **Image**: Ubuntu 24.04
   - **Type**:
     - **CPX21** (x86): 4 vCPU, 8 GB RAM (~€14/mo) – recommended
     - **CAX21** (ARM): 4 vCPU, 8 GB RAM (~€14/mo) – matches Oracle ARM setup
   - **SSH key**: Add your public key or create a new one
3. After creation, note the **public IP address**

---

## Step 2: Configure the Firewall

1. In Hetzner Cloud Console, go to **Firewalls** and click **Create Firewall**
2. Name it (e.g. `teslamate-firewall`)
3. Add **Inbound** rules:
   - **Port**: 22, **Protocol**: TCP, **Source**: Your IP or `0.0.0.0/0` (SSH)
   - **Port**: 80, **Protocol**: TCP, **Source**: `0.0.0.0/0` (HTTP)
   - **Port**: 443, **Protocol**: TCP, **Source**: `0.0.0.0/0` (HTTPS)
4. Click **Create Firewall**, then **Apply to** and select your new server

---

## Step 3: Point Your Domain to the Server

Create an **A record** for your subdomain (e.g. `teslamate.yourdomain.com`) pointing to the server's public IP.

---

## Step 4: Install Docker on the Server

SSH into your server:

```bash
ssh root@YOUR_PUBLIC_IP
```

Or, if you created a non-root user with your SSH key:

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

## Step 5: Deploy TeslaMate

1. Copy this project to the server (e.g. via `scp` or `git clone`):

```bash
# From your local machine
scp -r teslamate/ root@YOUR_PUBLIC_IP:~/
# Or: ssh root@YOUR_PUBLIC_IP "git clone https://github.com/YOUR_REPO/teslamate.git"
```

2. SSH into the server and edit `.env`:

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

## Step 6: Configure TeslaMate

1. Open `https://teslamate.yourdomain.com` in a browser (or on your phone)
2. Log in with the Basic Auth credentials (default: `teslamate` / `changeme`)
3. In **Settings**, set:
   - **Web App URL**: `https://teslamate.yourdomain.com`
   - **Grafana URL**: `https://teslamate.yourdomain.com/grafana`
4. Sign in with your Tesla account
5. Grafana dashboards: `https://teslamate.yourdomain.com/grafana` (use `GRAFANA_USER` / `GRAFANA_PW` from `.env`)

---

## Migrating from Oracle Cloud

If you have an existing TeslaMate deployment on Oracle Cloud and want to move it to Hetzner:

### 1. Backup on Oracle VM

```bash
cd ~/teslamate

# Backup PostgreSQL database
docker compose exec database pg_dump -U teslamate teslamate > teslamate_backup.sql

# Optional: backup Grafana data (dashboards, etc.)
docker compose run --rm -v $(pwd)/backup:/backup -v teslamate-grafana-data:/data alpine tar czf /backup/grafana-data.tar.gz -C /data .
```

Copy `teslamate_backup.sql` (and `backup/grafana-data.tar.gz` if created) to your local machine or directly to the new Hetzner server.

### 2. Create Hetzner Server

Follow **Steps 1–4** above to create the Hetzner server, configure the firewall, and install Docker.

### 3. Copy Project and Data to Hetzner

Copy the project, `.env`, `.htpasswd`, and the backup file(s) to the Hetzner server. Use the same `.env` values (especially `TM_ENCRYPTION_KEY`, `TM_DB_PASS`, `GRAFANA_PW`) so the database and Grafana work correctly.

### 4. Restore Data on Hetzner

```bash
cd ~/teslamate

# Start only the database first
docker compose up -d database

# Wait for PostgreSQL to be ready, then restore
sleep 10
docker compose exec -T database psql -U teslamate teslamate < teslamate_backup.sql

# Optional: restore Grafana data
# docker compose run --rm -v $(pwd)/backup:/backup -v teslamate-grafana-data:/data alpine tar xzf /backup/grafana-data.tar.gz -C /data

# Start the full stack
docker compose up -d
```

### 5. Update DNS

Point your domain's A record to the new Hetzner server IP. Let's Encrypt will issue new certificates automatically when Traefik starts.

### 6. Verify

1. Open `https://teslamate.yourdomain.com` and confirm your Tesla data is present
2. Check Grafana dashboards at `https://teslamate.yourdomain.com/grafana`
3. Once you're satisfied, you can shut down the Oracle VM

---

## Access from Phone

- **TeslaMate**: `https://teslamate.yourdomain.com` (Basic Auth, then Tesla login)
- **Grafana**: `https://teslamate.yourdomain.com/grafana`

Add these to your phone's home screen or bookmarks for quick access.

---

## Troubleshooting

### Let's Encrypt certificate fails

- Ensure ports 80 and 443 are open in the Hetzner Cloud Firewall
- Confirm your domain A record points to the server's public IP
- Check: `docker compose logs proxy`

### Cannot connect to Tesla API

- Ensure the server has outbound internet access
- Hetzner has no egress limits for traffic

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
