---
title: "Searxng Docker Misc"
date: 2026-05-09
draft: false
categories: ["docker", "self-hosted"]
tags: []
---

# SearXNG & Home Server Setup Summary

## Instance
- **URL**: `https://search.kartikpassbolt.org`
- **Host**: Raspberry Pi (aarch64), rootless Docker
- **Stack**: SearXNG, Valkey, Nginx Proxy Manager, AdGuard Home, Passbolt, Watchtower

---

## Chrome on iOS — SearXNG as Default Search Engine

Chrome on iOS doesn't support adding arbitrary custom search engines directly. It uses OpenSearch discovery instead.

**To add your instance:**
1. Visit `https://search.kartikpassbolt.org/search?q=test` in Chrome
2. Chrome will discover it via OpenSearch
3. Go to Chrome Settings > Search engine and set it as default

**If it disappears:** revisit the URL above and repeat. This is a Chrome iOS limitation — Firefox or Orion are better alternatives for a permanent fix.

**Search URL format:**
```
https://search.kartikpassbolt.org/search?q=%s
```

---

## Docker Compose — Final Configuration

Key changes from original:
- Replaced `containrrr/watchtower` (archived Dec 2025) with `nickfedor/watchtower`

```yaml
watchtower:
  container_name: watchtower
  image: nickfedor/watchtower:latest
  restart: always
  volumes:
    - /run/user/1000/docker.sock:/var/run/docker.sock
  command: --cleanup --schedule "0 0 4 * * *"
```

Watchtower runs daily at 4am, cleans up old images, and auto-updates all containers.

---

## Watchtower Troubleshooting

**Problem:** Watchtower crash-looping with `client version 1.25 is too old`

**Cause:** `containrrr/watchtower` is archived and its ARM64 image uses an outdated Docker client library incompatible with Docker API 1.40+.

**Fix:** Switch to the maintained fork:
```bash
# Update compose file to use nickfedor/watchtower:latest
docker compose up -d watchtower
docker logs watchtower --tail 10
```

**Verify it's working:**
```bash
# One-shot manual update test
docker run --rm \
  -v /run/user/1000/docker.sock:/var/run/docker.sock \
  nickfedor/watchtower:latest \
  --run-once
```

Expected output: `Update session completed` with `failed=0`

---

## Pi Health Checks

**Throttling check** — should return `0x0`:
```bash
vcgencmd get_throttled
```

**Check all containers running:**
```bash
docker ps -a
```

**Check container restart policy:**
```bash
docker inspect <container_name> | grep RestartPolicy
```

---

## Cron Jobs

```
0 5 * * 0    docker system prune -f           # Weekly Sunday 5am — clean unused images
0 6 1 * * *  /home/pi/renew-adguard-certs.sh  # Monthly 1st 6am — renew AdGuard SSL certs
```

**AdGuard cert renewal script** copies certs from NPM container into AdGuard container and restarts AdGuard. If it fails from cron but works manually, add to top of script:
```bash
export PATH=/usr/bin:/usr/local/bin:$PATH
```

---

## SearXNG Version Check

SearXNG uses rolling releases — check by image digest:
```bash
docker inspect searxng/searxng:latest | grep -E "Id|Created|RepoDigests"
```

Cross-reference the digest against Docker Hub to confirm you're on the latest build.

---

## AdGuard Home

**Setup:** Running in Docker with DoT (port 853) and DoH enabled. Plain DNS is disabled, forcing encrypted DNS only.

- **DoH:** `https://adguard.kartikpassbolt.org/dns-query`
- **DoT:** `adguard.kartikpassbolt.org` (port 853)
- **Admin panel:** `https://adguard.kartikpassbolt.org`
- **Certificate:** Let's Encrypt via NPM, auto-renewed monthly via cron

**Configure on Android:** Settings > Network > Private DNS > enter `adguard.kartikpassbolt.org`

**Configure on iPhone:** Settings > General > VPN & Device Management > DNS > add DoH profile with `https://adguard.kartikpassbolt.org/dns-query`

**Verify queries are flowing:**
Check `https://adguard.kartikpassbolt.org/#logs` — live DNS query log should show device traffic.

---

## Dynamic DNS

Not currently set up. Your public IP has been stable so far. If it changes, SearXNG and AdGuard will go offline until manually updated.

**To add Cloudflare DDNS** (if needed in future):
```yaml
ddns:
  container_name: cloudflare-ddns
  image: favonia/cloudflare-ddns:latest
  restart: always
  environment:
    - CF_API_TOKEN=your_token
    - DOMAINS=search.kartikpassbolt.org
    - PROXIED=false
```

**Check current IP vs DNS:**
```bash
curl -s https://api.ipify.org
dig search.kartikpassbolt.org +short
```

---

## SD Card Risk

Constant Docker I/O will wear out an SD card. Check if you're running off USB SSD:
```bash
lsblk
```

If root filesystem is on `/dev/mmcblk0` (SD card) rather than `/dev/sda` (USB), consider migrating Docker data to a USB SSD.
