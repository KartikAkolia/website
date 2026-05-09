---
title: "Trilium Setup"
date: 2026-05-09
draft: false
categories: ["self-hosted"]
tags: []
---

# Trilium Notes — Self-Hosted Setup (Raspberry Pi 5)

## Stack context

- Raspberry Pi 5 with NVMe, rootless Docker
- All containers on `searxng_default` network
- NPM handles reverse proxy + SSL, Cloudflare for DNS

---

## Docker Compose

```yaml
# ~/trilium/docker-compose.yml
services:
  trilium:
    image: zadam/trilium:latest
    container_name: trilium
    restart: unless-stopped
    volumes:
      - ./data:/home/node/trilium-data
    environment:
      - TRILIUM_DATA_DIR=/home/node/trilium-data
    networks:
      - searxng_default

networks:
  searxng_default:
    external: true
```

Port 8080 is internal only — NPM proxies by container name, no host port binding needed.

```bash
mkdir ~/trilium && cd ~/trilium
docker compose up -d
```

---

## Cloudflare DNS

- Type: `A`
- Name: `trilium`
- Value: server's public IP
- Proxy: enabled (orange cloud)

Trilium is plain HTTPS traffic, so Cloudflare proxying works fine. It hides your real IP and adds DDoS protection. DNS-only is for services using non-HTTP protocols or non-standard ports.

---

## NPM Proxy Host

| Field | Value |
|---|---|
| Domain | `trilium.kartikpassbolt.org` |
| Scheme | `http` |
| Forward Hostname | `trilium` |
| Forward Port | `8080` |
| Websockets Support | on |
| Block Common Exploits | on |
| SSL Certificate | Request new (Let's Encrypt) |
| Force SSL | on |
| HTTP/2 | on |

---

## Systemd User Service

```ini
# ~/.config/systemd/user/trilium.service
[Unit]
Description=Trilium Notes
After=searxng.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=%h/trilium
ExecStartPre=/bin/sleep 30
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable trilium.service
systemctl --user start trilium.service
```

---

## First Run

Navigate to `trilium.kartikpassbolt.org` → select **"I'm a new user"** → set password → database initialises. This screen only appears once.

---

## Importing Markdown

- Single file: right-click a note in the tree → Import → select `.md` file
- Directory: zip the folder first, then import the `.zip`
- Clipboard: editor block menu → paste markdown inline

## Exporting Markdown

- Single note: Note actions menu → Export
- Subtree: right-click note → Export → ZIP (mirrors tree structure)
- Protected notes: enter a protected session before exporting — output is unencrypted
