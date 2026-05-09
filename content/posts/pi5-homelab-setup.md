---
title: "Pi5 Homelab Setup"
date: 2026-05-09
draft: false
categories: ["raspberry-pi", "docker"]
tags: []
---

# Raspberry Pi 5 Homelab Setup Summary

## System Overview
- **Hardware:** Raspberry Pi 5 with NVMe SSD HAT
- **OS:** Raspberry Pi OS (Debian-based)
- **Docker:** Rootless Docker
- **Domain:** kartikpassbolt.org (Namecheap + Cloudflare DNS)
- **ISP:** Virgin Media, UK

---

## Running Containers

| Container | Image | Purpose |
|-----------|-------|---------|
| searxng-core | searxng/searxng | Private metasearch engine |
| searxng-valkey | valkey/valkey:9-alpine | SearXNG caching |
| nginx-proxy-manager | jc21/nginx-proxy-manager | Reverse proxy + SSL |
| adguardhome | adguard/adguardhome | DNS filtering + DoH/DoT |
| technitium | technitium/dns-server | Recursive DNS resolver |
| vaultwarden | vaultwarden/server | Password manager |
| watchtower | nickfedor/watchtower | Automatic container updates |

---

## Directory Structure

```
/home/pi/
├── searxng/          # SearXNG, Valkey, NPM, AdGuard, Watchtower
│   ├── docker-compose.yml
│   ├── .env
│   └── core-config/
│       └── settings.yml
├── vaultwarden/      # Vaultwarden
│   ├── docker-compose.yml
│   └── .env
├── technitium/       # Technitium DNS
│   └── docker-compose.yml
├── renew-adguard-certs.sh
└── renew-adguard-certs-dryrun.sh
```

---

## Docker Network

All containers share `searxng_default` network. Declare in every compose file:

```yaml
networks:
  searxng_default:
    external: true
```

For the main searxng stack, set as default network:

```yaml
networks:
  default:
    name: searxng_default
```

---

## ~/searxng/docker-compose.yml

```yaml
name: searxng
services:
  core:
    container_name: searxng-core
    image: docker.io/searxng/searxng:${SEARXNG_VERSION:-latest}
    restart: always
    ports:
      - ${SEARXNG_HOST:+${SEARXNG_HOST}:}${SEARXNG_PORT:-8080}:${SEARXNG_PORT:-8080}
    env_file: ./.env
    volumes:
      - ./core-config/:/etc/searxng/:Z
      - core-data:/var/cache/searxng/

  valkey:
    container_name: searxng-valkey
    image: docker.io/valkey/valkey:9-alpine
    command: valkey-server --save 30 1 --loglevel warning
    restart: always
    volumes:
      - valkey-data:/data/

  npm:
    container_name: nginx-proxy-manager
    image: jc21/nginx-proxy-manager:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - npm-data:/data
      - npm-letsencrypt:/etc/letsencrypt
    networks:
      - default

  adguard:
    container_name: adguardhome
    image: adguard/adguardhome:latest
    restart: always
    ports:
      - "0.0.0.0:53:53/tcp"
      - "0.0.0.0:53:53/udp"
      - "0.0.0.0:3000:3000"
      - "0.0.0.0:853:853"
      - "0.0.0.0:8853:8853"
    volumes:
      - adguard-work:/opt/adguardhome/work
      - adguard-conf:/opt/adguardhome/conf
    networks:
      - default

  watchtower:
    container_name: watchtower
    image: nickfedor/watchtower:latest
    restart: always
    volumes:
      - /run/user/1000/docker.sock:/var/run/docker.sock
    command: --cleanup --schedule "0 0 4 * * *"

networks:
  default:
    name: searxng_default

volumes:
  core-data:
  valkey-data:
  npm-data:
  npm-letsencrypt:
  adguard-work:
  adguard-conf:
```

---

## ~/vaultwarden/docker-compose.yml

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    volumes:
      - vaultwarden-data:/data
    env_file: ./.env
    environment:
      DOMAIN: "https://vault.kartikpassbolt.org"
      SIGNUPS_ALLOWED: "false"
      WEBSOCKET_ENABLED: "true"
      ADMIN_TOKEN: '$$argon2id$$v=19$$...'  # doubled $ signs for compose
    networks:
      - searxng_default

networks:
  searxng_default:
    external: true

volumes:
  vaultwarden-data:
```

**Generating Argon2 admin token:**
```bash
openssl rand -base64 48  # generate plain token
echo -n "your-token" | argon2 "$(openssl rand -base64 32)" -id -k 65540 -t 3 -p 4 -l 32 -e
```
Use plain token to log into `/admin`. Double all `$` in compose file.

---

## ~/technitium/docker-compose.yml

```yaml
services:
  technitium:
    image: technitium/dns-server:latest
    container_name: technitium
    restart: always
    ports:
      - "5380:5380"
    volumes:
      - technitium-data:/etc/dns
    environment:
      DNS_SERVER_DOMAIN: "dns.kartikpassbolt.org"
      DNS_SERVER_PREFER_IPV6: "false"
      DNS_SERVER_RECURSION: "Allow"
      DNS_SERVER_RECURSION_DENIED_NETWORKS: "0.0.0.0/0"
      DNS_SERVER_RECURSION_ALLOWED_NETWORKS: "172.18.0.0/16,127.0.0.1"
      DNS_SERVER_ENABLE_BLOCKING: "false"
      DNS_SERVER_CACHE_MINIMUM_RECORD_TTL: "300"
      DNS_SERVER_CACHE_MAXIMUM_RECORD_TTL: "86400"
      DNS_SERVER_CACHE_FAILED_NEGATIVE_TTL: "10"
      DNS_SERVER_CACHE_PREFETCH_ELIGIBLE_COUNT: "1"
      DNS_SERVER_CACHE_PREFETCH_TRIGGER: "9"
      DNS_SERVER_CACHE_PREFETCH_SAMPLE_INTERVAL: "5"
      DNS_SERVER_CACHE_PREFETCH_SAMPLE_ELIGIBILITY_HIT_COUNT: "30"
    networks:
      - searxng_default

networks:
  searxng_default:
    external: true

volumes:
  technitium-data:
```

**AdGuard upstream DNS:** Set to `172.18.0.8:53` (check with `docker inspect technitium | grep IPAddress`)

---

## Systemd Services

Location: `~/.config/systemd/user/`

### searxng.service
```ini
[Unit]
Description=SearXNG Stack
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/pi/searxng
ExecStartPre=/bin/sleep 20
ExecStartPre=/usr/bin/sudo /sbin/sysctl -p /etc/sysctl.d/99-unprivileged-ports.conf
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

### vaultwarden.service
```ini
[Unit]
Description=Vaultwarden Stack
Requires=docker.service
After=docker.service searxng.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/pi/vaultwarden
ExecStartPre=/bin/sleep 30
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

### technitium.service
```ini
[Unit]
Description=Technitium DNS Stack
Requires=docker.service
After=docker.service searxng.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/pi/technitium
ExecStartPre=/bin/sleep 25
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

**Enable services:**
```bash
systemctl --user daemon-reload
systemctl --user enable searxng.service vaultwarden.service technitium.service
```

---

## Sysctl - Unprivileged Ports

Required for rootless Docker to bind ports below 1024:

```bash
echo "net.ipv4.ip_unprivileged_port_start=53" | sudo tee /etc/sysctl.d/99-unprivileged-ports.conf
```

**Sudoers entry** (for systemd service):
```
pi ALL=(ALL) NOPASSWD: /sbin/sysctl -p /etc/sysctl.d/99-unprivileged-ports.conf
```

---

## UFW Firewall Rules

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp   # SSH
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS
sudo ufw allow 53/tcp     # DNS
sudo ufw allow 53/udp     # DNS
sudo ufw allow 853/tcp    # DoT
sudo ufw allow 81/tcp     # NPM admin (LAN only)
sudo ufw allow 3000/tcp   # AdGuard setup
sudo ufw allow 5380/tcp   # Technitium UI
sudo ufw enable
```

---

## SSH Hardening

```bash
# Generate key on Windows
ssh-keygen -t ed25519 -C "your@email.com"
# Copy to Pi
type C:\Users\Kartik\.ssh\id_ed25519.pub | ssh pi@192.168.0.166 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# Fix permissions
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

`/etc/ssh/sshd_config`:
```
Port 2222
PasswordAuthentication no
PubkeyAuthentication yes
```

---

## Fail2ban

`/etc/fail2ban/jail.local`:
```
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = 2222
```

---

## Cloudflare DNS Records

| Type | Name | Content | Proxy |
|------|------|---------|-------|
| A | search | 86.2.4.121 | DNS only |
| A | adguard | 86.2.4.121 | DNS only |
| A | vault | 86.2.4.121 | DNS only |
| A | mc | 86.2.4.121 | DNS only |

---

## NPM Proxy Hosts

| Domain | Forward Host | Port | SSL |
|--------|-------------|------|-----|
| search.kartikpassbolt.org | searxng-core | 8080 | Let's Encrypt |
| adguard.kartikpassbolt.org | adguardhome | 3000 | Let's Encrypt |
| vault.kartikpassbolt.org | vaultwarden | 80 | Let's Encrypt |

AdGuard custom location: `/dns-query` → `adguardhome:8853` (https)

---

## AdGuard Cert Renewal Script

`~/renew-adguard-certs.sh`:
```bash
#!/bin/bash
echo "Copying AdGuard certificates from NPM..."

NPM_NUM=$(docker exec nginx-proxy-manager grep -r "adguard.kartikpassbolt.org" /etc/letsencrypt/renewal/ | grep -o 'npm-[0-9]*' | head -1)
[ -z "$NPM_NUM" ] && echo "ERROR: Could not find npm cert folder" && exit 1
echo "Found cert folder: $NPM_NUM"

FULLCHAIN=$(docker exec nginx-proxy-manager find /etc/letsencrypt/archive/$NPM_NUM -name 'fullchain*.pem' | sort -V | tail -1)
[ -z "$FULLCHAIN" ] && echo "ERROR: Could not find fullchain.pem" && exit 1

PRIVKEY=$(docker exec nginx-proxy-manager find /etc/letsencrypt/archive/$NPM_NUM -name 'privkey*.pem' | sort -V | tail -1)
[ -z "$PRIVKEY" ] && echo "ERROR: Could not find privkey.pem" && exit 1

rm -f ~/searxng/adguard-fullchain.pem ~/searxng/adguard-privkey.pem

docker cp nginx-proxy-manager:$FULLCHAIN ~/searxng/adguard-fullchain.pem
docker cp nginx-proxy-manager:$PRIVKEY ~/searxng/adguard-privkey.pem
docker cp ~/searxng/adguard-fullchain.pem adguardhome:/opt/adguardhome/conf/fullchain.pem
docker cp ~/searxng/adguard-privkey.pem adguardhome:/opt/adguardhome/conf/privkey.pem

docker restart adguardhome

rm -f ~/searxng/adguard-fullchain.pem ~/searxng/adguard-privkey.pem
echo "Done. AdGuard certificates updated."
```

**Crontab:**
```
0 5 * * 0 docker system prune -f
0 6 1 * * /home/pi/renew-adguard-certs.sh
```

---

## Adding a New Container (Template)

1. Create directory: `mkdir ~/appname && cd ~/appname`
2. Create `docker-compose.yml` with `searxng_default` network
3. If public: add Cloudflare A record + NPM proxy host with SSL
4. If LAN only: add UFW rule for port
5. `docker compose up -d`
6. Create systemd service in `~/.config/systemd/user/appname.service`
7. `systemctl --user daemon-reload && systemctl --user enable appname.service`

---

## Common Troubleshooting

**Containers not starting after reboot:**
```bash
cat /proc/sys/net/ipv4/ip_unprivileged_port_start  # should be 53
sudo sysctl -p /etc/sysctl.d/99-unprivileged-ports.conf
systemctl --user restart searxng.service
```

**NPM can't reach container:**
```bash
docker network connect searxng_default <container-name>
```

**Check all container status:**
```bash
docker ps -a
systemctl --user status searxng.service
systemctl --user status vaultwarden.service
systemctl --user status technitium.service
```

**Technitium IP changed after restart:**
```bash
docker inspect technitium | grep IPAddress
# Update AdGuard upstream DNS with new IP
```

**Docker prune manually:**
```bash
docker system prune -f
```

---

## Watchtower Schedule

Runs daily at 4am, updates all containers and removes old images automatically.

---

## Vaultwarden Admin Access

URL: `https://vault.kartikpassbolt.org/admin`
Use the original plain text token (not the Argon2 hash) to log in.

---

## DNS Stack Flow

```
Device → AdGuard (port 53, filtering) → Technitium (172.18.0.8:53, recursive) → Root servers
```

DoH for browsers: `https://adguard.kartikpassbolt.org/dns-query`
DoT for Android: `adguard.kartikpassbolt.org` (port 853)
