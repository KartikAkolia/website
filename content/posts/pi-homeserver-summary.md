---
title: "Pi Homeserver Summary"
date: 2026-05-09
draft: false
categories: ["raspberry-pi", "docker"]
tags: []
---

# Raspberry Pi 5 Home Server — Session Summary

## Stack
- SearXNG (search) — `search.kartikpassbolt.org`
- AdGuard Home (DNS/filtering) — `adguard.kartikpassbolt.org`
- Passbolt (password manager) — `passbolt.kartikpassbolt.org`
- Vaultwarden (passwords, local)
- Nginx Proxy Manager (reverse proxy)
- Watchtower (auto container updates)
- Valkey (SearXNG cache)
- MariaDB (Passbolt database)

---

## NPM Hardening

### Applied to all proxy hosts
- Block Common Exploits: **on**
- Force SSL: **on**
- HTTP/2 Support: **on**

### Passbolt & AdGuard Home
- Access List: restricted to known IPs only (not publicly accessible)

---

## SSH Hardening (`/etc/ssh/sshd_config`)
```
Port 2222
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
MaxAuthTries 3
LoginGraceTime 30
```
Restart after changes:
```bash
sudo systemctl restart ssh
```

---

## UFW
- Port 53 not forwarded on router
- Only ports 80, 443, 2222 (SSH) open
- Fail2ban monitoring SSH

---

## Disabled Services
```bash
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now avahi-daemon.socket
sudo systemctl disable --now bluetooth
```
**Note:** Disabling avahi breaks `raspberrypi.local` hostname resolution. Re-enable if needed:
```bash
sudo systemctl enable --now avahi-daemon
sudo systemctl enable --now avahi-daemon.socket
```

---

## DNS
Changed from ISP DNS to Cloudflare on the Pi host:
```bash
nmcli connection show  # find connection name
sudo nmcli connection modify "Wired connection 1" ipv4.dns "1.1.1.1 1.0.0.1"
sudo nmcli connection modify "Wired connection 1" ipv4.ignore-auto-dns yes
sudo nmcli connection up "Wired connection 1"
cat /etc/resolv.conf  # verify
```

---

## sysctl (`/etc/sysctl.conf`)
```
net.ipv4.ip_unprivileged_port_start=53
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Security
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_rfc1337 = 1
net.ipv4.tcp_fin_timeout = 15

# Performance
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
```
Apply:
```bash
sudo sysctl -p
```

### BBR Congestion Control
```bash
sudo modprobe tcp_bbr
sysctl net.ipv4.tcp_available_congestion_control  # verify bbr listed
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
echo "tcp_bbr" | sudo tee -a /etc/modules  # persist across reboots
```

---

## Docker (`/etc/docker/daemon.json`)
```json
{
  "dns": ["1.1.1.1", "1.0.0.1"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true
}
```
Apply:
```bash
sudo systemctl restart docker
```

---

## cgroup Support (removes Docker memory/swap warnings)
Add to end of first line in `/boot/firmware/cmdline.txt` (must remain a single line):
```
cgroup_enable=memory cgroup_memory=1 swapaccount=1
```
Then reboot.

---

## Docker Network Troubleshooting
After reboot, containers lost network memberships. Root cause: containers were on default bridge network, which doesn't support hostname resolution between containers.

**Fix:** Add shared named network to docker-compose files:
```yaml
networks:
  default:
    name: searxng_default
```
This ensures all containers join the same named network and can resolve each other by hostname across reboots.

**Temporary fix used (revert after proper fix):**
```bash
# Find container IP
docker inspect adguardhome --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# Edit NPM config and replace hostname with IP
sudo nano /home/pi/.local/share/docker/volumes/searxng_npm-data/_data/nginx/proxy_host/2.conf
docker restart nginx-proxy-manager
```

---

## Unattended Upgrades
Already installed and active. Verify:
```bash
systemctl status unattended-upgrades
cat /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## Performance Testing (iperf3)
```bash
# On Pi (server)
sudo ufw allow 5201
iperf3 -s

# On another machine (client)
iperf3 -c 192.168.0.166 -V -t 30 -P 4

# Cleanup
sudo ufw delete allow 5201
```
Result: ~583 Mbits/sec, 4.4% CPU, BBR confirmed active.

---

## Containers to Consider Adding
- **Uptime Kuma** — service monitoring and alerts
- **wg-easy** — WireGuard VPN server with web UI
- **Nextcloud** — Dropbox replacement (heavier, check RAM first)
- **Syncthing** — lightweight file sync, peer-to-peer
- **Portainer** — Docker management UI
- **Stirling PDF** — local PDF tools

### Check available RAM before adding containers
```bash
free -h
```

---

## wg-easy (WireGuard VPN) Setup
Generate password hash:
```bash
docker run --rm -it ghcr.io/wg-easy/wg-easy wgpw YOUR_PASSWORD
```

`docker-compose.yml`:
```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    environment:
      - LANG=en
      - WG_HOST=kartikpassbolt.org
      - PASSWORD_HASH=<bcrypt_hash>
      - WG_PORT=51820
      - PORT=51821
    volumes:
      - etc_wireguard:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
    networks:
      default:
        name: searxng_default

volumes:
  etc_wireguard:
```
Requires UDP port 51820 forwarded on router. Proxy the web UI (port 51821) through NPM.
