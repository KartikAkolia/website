---
title: "Kartikpassbolt Security Setup"
date: 2026-05-09
draft: false
categories: ["security"]
tags: []
---

# kartikpassbolt.org — Domain & Security Setup Summary

## What Was Configured

### Email Security (DNS Records)
- SPF, DKIM, and DMARC records configured via Cloudflare DNS
- Zoho Mail used as the email provider (MX records pointing to `mx.zoho.eu`, `mx2.zoho.eu`, `mx3.zoho.eu`)
- Resend DKIM key (`resend._domainkey`) removed after discontinuing Resend as a sending service

### DNS Records in Place

| Type | Name | Purpose |
|---|---|---|
| TXT | `kartikpassbolt.org` | SPF — `v=spf1 include:zohomail.eu ~all` |
| TXT | `zmail._domainkey` | DKIM for Zoho Mail |
| TXT | `_dmarc` | DMARC — `v=DMARC1; p=quarantine` |
| MX | `kartikpassbolt.org` | Zoho Mail routing |

---

## Cloudflare SSL/TLS Configuration

### Encryption Mode
Set to **Full (Strict)** via Custom SSL/TLS (automatic mode disabled).

Path: `SSL/TLS > Overview > Configure`

### Edge Certificates
- Universal SSL certificate active (managed by Cloudflare)
- Expires: 2026-06-17

### Settings Applied

Path: `SSL/TLS > Edge Certificates`

| Setting | Value |
|---|---|
| Always Use HTTPS | On |
| Minimum TLS Version | TLS 1.2 |
| HSTS | Enabled |
| HSTS Max-Age | 6 months (recommended) |
| HSTS Include Subdomains | On |
| HSTS Preload | Off |
| No-Sniff Header | On |

> **Note:** HSTS subdomains is safe to enable because all proxied subdomains have valid Let's Encrypt certs via NPM. `mc.kartikpassbolt.org` is DNS only (Minecraft, no HTTPS) and is not proxied, so it is unaffected by HSTS.

### SSL Labs Result
All Cloudflare IPs scored **A+** for `vault.kartikpassbolt.org`. HSTS is what pushes the grade from A to A+.

---

## Cloudflare Proxy — Subdomains

All subdomains below are proxied through Cloudflare (orange cloud). Origin IP is hidden; traffic hits Cloudflare's network first.

| Subdomain | Service | SSL |
|---|---|---|
| `adguard.kartikpassbolt.org` | AdGuard Home | Let's Encrypt via NPM |
| `search.kartikpassbolt.org` | SearXNG | Let's Encrypt via NPM |
| `trilium.kartikpassbolt.org` | Trilium Notes | Let's Encrypt via NPM |
| `vault.kartikpassbolt.org` | Vaultwarden | Let's Encrypt via NPM |
| `mc.kartikpassbolt.org` | Minecraft | DNS only — no proxy, no HTTPS |

### Verifying Origin IP is Hidden
```bash
nslookup vault.kartikpassbolt.org
```
Expected: Cloudflare IP ranges only (e.g. `104.21.x.x`, `172.67.x.x`). Your server IP should not appear.

---

## Nginx Proxy Manager (NPM)

Running in Docker on Raspberry Pi 5. Handles SSL termination for all proxied subdomains.

- All four active subdomains have valid Let's Encrypt certs
- Force SSL disabled on adguard and search (Cloudflare handles HTTP→HTTPS redirect via Always Use HTTPS)
- Vault and Trilium never had Force SSL enabled

Traffic chain: `Browser → Cloudflare (HTTPS) → NPM (HTTPS) → Internal service`

> If you re-enable Force SSL in NPM on any subdomain that is also proxied through Cloudflare, you may hit redirect loops. Let Cloudflare handle the redirect.

---

## Raspberry Pi Hardening

| Item | Status |
|---|---|
| SSH authentication | Public/private key only — password auth disabled |
| SSH exposure | Not exposed to the internet |
| Firewall (UFW) | Enabled — only required ports open |
| OS updates | Automatic |
| Container updates | Watchtower running — pulls updated images on schedule |
| Storage | NVMe drive (more reliable than SD card) |

---

## Vaultwarden Backup (Pending Setup)

All vault data for all users lives in the Vaultwarden data directory (SQLite database + attachments).

### Planned approach
- External SATA enclosure connected via USB to the Pi
- Periodic backup of the Vaultwarden data directory to the external drive

### Recommended backup container
```
bruceforce/vaultwarden-backup
```
Handles scheduled backups automatically. Stop the Vaultwarden container briefly during backup to avoid mid-write SQLite corruption, then restart.

> Set this up before adding family members as users on the instance.

---

## Useful Checks

| Check | Tool |
|---|---|
| SSL grade | https://www.ssllabs.com/ssltest/ |
| Security headers | https://securityheaders.com |
| DMARC reports | Cloudflare > Email > DMARC Management |
| Origin IP hidden | `nslookup <subdomain>` — should return Cloudflare IPs only |
| MX / email health | https://mxtoolbox.com/emailhealth |
