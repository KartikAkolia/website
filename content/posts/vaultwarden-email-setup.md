---
title: "Vaultwarden Email Setup"
date: 2026-05-09
draft: false
categories: ["self-hosted", "security"]
tags: []
---

# Vaultwarden Email Troubleshooting & Gmail SMTP Setup

## Problem

Invited user (`user@gmail.com`) never received the invite email. The user showed as "Invited" and "Verified" in the admin panel but "Last Active: Never."

## Diagnosis

- SMTP was working (admin could send to themselves)
- Server IP (`86.2.4.121`) was listed on **Spamhaus ZEN / PBL (Policy Blocklist)**
- PBL lists residential/broadband IPs that shouldn't be sending mail directly
- Gmail checks Spamhaus, so emails from this IP get rejected silently

## Fix: Gmail SMTP Relay

Route Vaultwarden's emails through Gmail instead of sending directly from the server IP.

### 1. Create a Gmail App Password

1. Go to [myaccount.google.com](https://myaccount.google.com)
2. Security > 2-Step Verification (enable if not already)
3. Search "App passwords" in the search bar
4. Create one, name it "Vaultwarden"
5. Copy the 16-character code

### 2. Update `docker-compose.yml`

Add the following under `environment`:

```yaml
SMTP_HOST: "smtp.gmail.com"
SMTP_PORT: "587"
SMTP_SECURITY: "starttls"
SMTP_USERNAME: "yourgmail@gmail.com"
SMTP_PASSWORD: "your-app-password"
SMTP_FROM: "yourgmail@gmail.com"
```

### 3. Restart Vaultwarden

```bash
docker compose down && docker compose up -d
```

### 4. Resend the invite

In the Vaultwarden admin panel (Users), click **Resend invite** for the affected user.

## Useful Checks

- Check if your IP is blacklisted: [mxtoolbox.com/blacklists.aspx](https://mxtoolbox.com/blacklists.aspx)
- Check Spamhaus listing details: [check.spamhaus.org](https://check.spamhaus.org)
- Check Vaultwarden SMTP logs: `docker logs -f vaultwarden 2>&1 | grep -i smtp`
