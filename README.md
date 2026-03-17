<div align="center">
  <img src="https://raw.githubusercontent.com/teslamate-org/teslamate/main/website/static/img/logo.svg" alt="TeslaMate" width="120" />
  <h1>TeslaMate – Cloud Deployment</h1>
  <p>Self-hosted Tesla data logging with HTTPS and public access</p>
  <p>
    <img src="https://img.shields.io/docker/v/teslamate/teslamate/latest?label=TeslaMate" alt="TeslaMate version" />
    <img src="https://img.shields.io/docker/pulls/teslamate/teslamate?color=%23099cec" alt="Docker pulls" />
    <img src="https://img.shields.io/badge/Oracle%20Cloud-Free%20tier-green" alt="Oracle Cloud" />
    <img src="https://img.shields.io/badge/Hetzner%20Cloud-supported-blue" alt="Hetzner Cloud" />
  </p>
</div>

---

## What is TeslaMate?

[TeslaMate](https://teslamate.org/) is a self-hosted data logger for your Tesla vehicle. It logs driving data, charging sessions, and vehicle state, and provides beautiful Grafana dashboards for visualization. The official documentation is at [docs.teslamate.org](https://docs.teslamate.org/).

![TeslaMate Web Interface](https://raw.githubusercontent.com/teslamate-org/teslamate/main/website/static/screenshots/web_interface.png)

*TeslaMate web interface with Grafana dashboards*

---

## What This Repo Provides

This is not a fork of TeslaMate—it uses the official `teslamate/teslamate` and `teslamate/grafana` Docker images and adds:

- **Traefik** reverse proxy with automatic Let's Encrypt HTTPS
- **Basic Auth** for TeslaMate and Grafana
- **Public access** from phone or any device (not just home network)
- Pre-configured `docker-compose.yml` with TeslaMate, PostgreSQL, Grafana, Mosquitto (MQTT), and Traefik
- Cloud-agnostic setup (ARM and x86 compatible, minimal resources)

---

## Architecture

```mermaid
flowchart LR
    subgraph Internet
        User[User / Phone]
    end
    subgraph Server
        Traefik[Traefik Proxy]
        TeslaMate[TeslaMate]
        Grafana[Grafana]
        Postgres[(PostgreSQL)]
        Mosquitto[Mosquitto]
    end
    User -->|HTTPS| Traefik
    Traefik --> TeslaMate
    Traefik --> Grafana
    TeslaMate --> Postgres
    TeslaMate --> Mosquitto
```

---

## Deployment Options

| Cloud | Guide | Notes |
|-------|-------|-------|
| **Oracle Cloud** | [ORACLE_CLOUD_DEPLOYMENT.md](ORACLE_CLOUD_DEPLOYMENT.md) | Free tier, ARM (Ampere) |
| **Hetzner Cloud** | [HETZNER_CLOUD_DEPLOYMENT.md](HETZNER_CLOUD_DEPLOYMENT.md) | ~€14/mo, x86 or ARM, includes migration from Oracle |

---

## Quick Start

> **Prerequisites:** A cloud VM (Oracle or Hetzner), a domain name pointing to your server, Docker and Docker Compose.

1. Clone this repo and `cd` into it.
2. Copy `.env.example` to `.env` and fill in your values.
3. Copy `.htpasswd.example` to `.htpasswd` and change the default password before going live.
4. Run `docker compose up -d`.

For a full step-by-step guide including VM creation, domain setup, and troubleshooting, see the deployment guide for your cloud provider above.

---

## Configuration

Required variables in `.env`:

| Variable | Description |
|----------|-------------|
| `TM_ENCRYPTION_KEY` | Run `openssl rand -base64 32` and paste the result |
| `TM_DB_PASS` | Strong PostgreSQL password |
| `GRAFANA_PW` | Grafana admin password |
| `FQDN_TM` | Your domain (e.g. `teslamate.yourdomain.com`) |
| `LETSENCRYPT_EMAIL` | Your email for Let's Encrypt |

Generate a `.htpasswd` with your own password:

```bash
htpasswd -nbB teslamate YOUR_PASSWORD > .htpasswd
```

---

## Security Checklist

- [ ] Replace `TM_ENCRYPTION_KEY` with a strong random key
- [ ] Use strong `TM_DB_PASS` and `GRAFANA_PW`
- [ ] Change `.htpasswd` password from default `changeme`
- [ ] Keep `.env` and `.htpasswd` out of version control (they are in `.gitignore`)

---

## Links

- [Oracle Cloud deployment guide](ORACLE_CLOUD_DEPLOYMENT.md)
- [Hetzner Cloud deployment guide](HETZNER_CLOUD_DEPLOYMENT.md)
- [TeslaMate documentation](https://docs.teslamate.org/)
- [TeslaMate GitHub](https://github.com/teslamate-org/teslamate)
