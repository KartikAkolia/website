# Kartik Homelab Website

Personal homelab documentation site built with Hugo and the Clarity theme.
Serves as a cheat sheet for replicating my Raspberry Pi 5 homelab setup from scratch.

## Stack

- Hugo v0.161.1 (extended)
- Theme: [hugo-clarity](https://github.com/chipzoller/hugo-clarity)
- Served via nginx (Docker)
- Reverse proxy: Nginx Proxy Manager
- DNS: Cloudflare

## Content

- Pi5 Homelab Setup
- SearXNG Docker
- AdGuard Home
- Raspberry Pi SSH Setup
- Vaultwarden Email Setup
- Kartikpassbolt Security Setup
- Kubernetes Setup
- Trilium Setup
- Uptime Kuma Setup

## Usage

Build the site:
```bash
cd ~/hugo/mysite
hugo --minify
```

Local dev server:
```bash
hugo server -D --bind 0.0.0.0 --baseURL http://192.168.0.166:1313
```

## Deployment

The `public/` directory is served by an nginx container defined in `~/hugosite/docker-compose.yml`.
Rebuild with `hugo --minify` and changes are live immediately via the volume mount.
