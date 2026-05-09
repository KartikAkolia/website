---
title: "RaspberryPi SSH Setup Summary"
date: 2026-05-09
draft: false
categories: ["security", "raspberry-pi"]
tags: []
---

# Raspberry Pi SSH & DNS Setup Summary

## PuTTY SSH Key Authentication

### Convert existing OpenSSH key to PuTTY format
1. Open **PuTTYgen**
2. Click **Load** → set file filter to "All Files" → select `C:\Users\Kartik\.ssh\id_ed25519`
3. Click **Save private key** → save as `C:\Users\Kartik\.ssh\raspberrypi.ppk`

### Point PuTTY to the key
1. Open PuTTY → load your **RaspberryPi** session
2. Go to **Connection → SSH → Auth → Credentials**
3. Browse to `C:\Users\Kartik\.ssh\raspberrypi.ppk`
4. Go back to **Session** → click **Save**

### Copy public key to the Pi (run once from PowerShell)
```powershell
type C:\Users\Kartik\.ssh\id_ed25519.pub | ssh kartikakolia@192.168.0.166 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

## Pi SSH Configuration (`/etc/ssh/sshd_config`)

Key settings confirmed active:

- `Port 2222` — PuTTY must be set to port **2222**, not 22
- `PasswordAuthentication no` — password login is disabled
- `KbdInteractiveAuthentication no` — keyboard-interactive auth disabled
- `PubkeyAuthentication yes` — key-based auth is enabled

> If the public key isn't in `~/.ssh/authorized_keys` on the Pi and password auth is off, you'll need a monitor + keyboard to get in.

---

## Technitium DNS Docker Compose

### Issues with current config
- Port 53 (UDP/TCP) is missing — DNS traffic won't work without it
- Tied to `searxng_default` network — if SearXNG goes down, DNS goes with it
- `restart: always` restarts even manually stopped containers

### Recommended `docker-compose.yml`
```yaml
services:
  technitium:
    image: technitium/dns-server:latest
    container_name: technitium
    restart: unless-stopped
    network_mode: host
    volumes:
      - technitium-data:/etc/dns
    environment:
      DNS_SERVER_DOMAIN: "dns.kartikpassbolt.org"

volumes:
  technitium-data:
```

### Key changes
- `network_mode: host` — lower latency, skips Docker NAT, removes need for explicit port mappings
- Removed `searxng_default` network dependency — DNS runs independently
- `restart: unless-stopped` — won't auto-restart a deliberately stopped container
- Ports 53/UDP, 53/TCP, and 5380 are all accessible via host mode automatically
