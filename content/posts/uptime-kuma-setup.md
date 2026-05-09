---
title: "Uptime Kuma Setup"
date: 2026-05-09
draft: false
categories: ["self-hosted"]
tags: []
---

# Uptime Kuma Setup — Raspberry Pi 5

## Stack context

Raspberry Pi 5, rootless Docker, all containers on `searxng_default` network. Domain: `kartikpassbolt.org` on Cloudflare.

---

## Files

### `~/uptime-kuma/docker-compose.yml`

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
      - /run/user/1000/docker.sock:/var/run/docker.sock:ro
    networks:
      - searxng_default

networks:
  searxng_default:
    external: true
```

Note: `/run/user/1000/docker.sock` is required for rootless Docker. The standard `/var/run/docker.sock` will fail with `EACCES`.

### `~/.config/systemd/user/uptime-kuma.service`

```ini
[Unit]
Description=Uptime Kuma
Requires=searxng.service
After=searxng.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/sleep 30
ExecStart=/usr/bin/docker compose -f %h/uptime-kuma/docker-compose.yml up -d
ExecStop=/usr/bin/docker compose -f %h/uptime-kuma/docker-compose.yml down
WorkingDirectory=%h/uptime-kuma

[Install]
WantedBy=default.target
```

---

## Setup commands

```bash
mkdir ~/uptime-kuma
cd ~/uptime-kuma
# create docker-compose.yml as above
docker compose up -d
docker ps | grep uptime-kuma

systemctl --user daemon-reload
systemctl --user enable uptime-kuma.service
```

UFW rule (LAN only, no public exposure):

```bash
sudo ufw allow from 192.168.0.0/24 to any port 3001
sudo ufw reload
```

Access at: `http://192.168.0.166:3001`

---

## Troubleshooting

**`connect EACCES /var/run/docker.sock`** — rootless Docker uses a different socket. Verify with:

```bash
echo $DOCKER_HOST
ls /run/user/$(id -u)/docker.sock
```

Fix: mount `/run/user/1000/docker.sock` instead of `/var/run/docker.sock` in the compose file, then:

```bash
docker compose up -d --force-recreate
```

Data in `./data` volume is preserved on recreate.

---

## Monitors

### HTTP monitors

| Name | URL |
|------|-----|
| SearXNG | `https://search.kartikpassbolt.org` |
| AdGuard Home | `https://adguard.kartikpassbolt.org` |
| Vaultwarden | `https://vault.kartikpassbolt.org` |
| NPM (HTTP) | `http://192.168.0.166:81` |
| Uptime Kuma | `http://192.168.0.166:3001` |
| Technitium DNS | `http://192.168.0.166:5380` |

### Docker container monitors

| Name | Container Name |
|------|---------------|
| SearXNG | `searxng-core` |
| Valkey | `searxng-valkey` |
| Vaultwarden | `vaultwarden` |
| NPM (Docker) | `nginx-proxy-manager` |
| AdGuard Home | `adguardhome` |
| Technitium DNS | `technitium` |
| Watchtower | `watchtower` |
| Uptime Kuma | `uptime-kuma` |

Docker host setup in UI: Friendly Name `local`, Connection Type `Socket`, Docker Daemon `/var/run/docker.sock`.

---

## Notes

- Monitors are stored in SQLite — cannot be defined in compose file. UI or `uptime-kuma-api` (Python) only.
- HTTP authentication field: leave as `None` for all services — they handle auth in their own UI.
- NPM appears twice in the monitor list by design: one HTTP check, one Docker container check. Rename them to avoid confusion.
