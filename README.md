# Riot Homelab

A self-hosted home lab built from the ground up on a borrowed PC — running a media server, photo backup system, network-wide ad blocking, remote access, and service monitoring.

This started as a personal project to stop paying for streaming subscriptions. It grew into something I'm genuinely proud of, a real infrastructure project that taught me Linux, Docker, networking, bash scripting, and systems thinking in a way that no tutorial ever could.

---

## Why I Built This

I had a few specific problems:

- **Media** — paying for multiple streaming services for content I already owned on DVD or could source myself. Wanted a self-hosted Plex/Jellyfin alternative that I fully controlled.
- **Photos** — my girlfriend had 42,000 photos and videos on her phone with no backup strategy. Losing those was not acceptable. Didn't want to hand her memories to Google or Apple.
- **Privacy** — I wanted to own my infrastructure. My data, my hardware, my rules.

These aren't abstract problems. They're real, they affected real people I care about, and solving them required building something real.

---

## What's Running

### Infrastructure
- **OS** — Ubuntu 24.04 LTS, dual-booted alongside Windows on a loaned PC
- **Remote Access** — Tailscale (WireGuard-based VPN) for secure access from anywhere with zero port forwarding
- **Remote Desktop** — XRDP + XFCE for GUI management from my own machine
- **Reverse Proxy** — Nginx Proxy Manager with friendly local DNS names via Pi-hole
- **Container Runtime** — Docker + Docker Compose for all services

### Services
| Service | Purpose | Port |
|---|---|---|
| Jellyfin | Media server — movies, TV shows | 8096 |
| Immich | Self-hosted Google Photos alternative | 2283 |
| Pi-hole | Network-wide DNS ad blocker + local DNS | 8080 |
| Nginx Proxy Manager | Reverse proxy + SSL management | 81 |
| Portainer | Docker container management UI | 9000 |
| Uptime Kuma | Service monitoring + Discord alerts | 3001 |
| Glance | Unified homelab dashboard | 8585 |
| Watchtower | Automatic container updates (4AM daily) | — |
| Vaultwarden | Self-hosted Bitwarden password manager | 8181 |
| TeamSpeak 3 | Voice comms server (TFAR mod support) | 9987 |

---

## Architecture

See [`architecture.html`](./docs/architecture.html) for a visual diagram of the full stack.

The short version:

```
Internet
  └── Altice ISP Gateway
        └── Ubuntu Server (192.168.1.200)
              ├── Docker
              │     ├── Jellyfin
              │     ├── Immich → 1TB external USB
              │     ├── Pi-hole (DNS)
              │     ├── Nginx Proxy Manager
              │     ├── Portainer
              │     ├── Uptime Kuma
              │     ├── Glance
              │     ├── Watchtower
              │     ├── Vaultwarden
              │     └── TeamSpeak 3

Remote Access: Tailscale mesh VPN (all devices)
Local DNS: Pi-hole (*.riot-homelab)
```

---

## Interesting Technical Challenges

A few problems that weren't solved by copy-pasting a tutorial:

**Immich storage overflow** — 42,000 photos and videos exceeded available SSD space mid-sync. Solved by reformatting a spare 1TB external drive as ext4, mounting it persistently via fstab with the `nofail` flag, and live-migrating Immich's data directory by updating the Docker Compose volume mapping and restarting the container without data loss.

---

## What I Learned

I came into this knowing Python and some programming fundamentals. I left with practical experience in:

- Linux system administration (Ubuntu, systemd, cron, file permissions, users)
- Docker and Docker Compose (containers, volumes, networks, multi-service stacks)
- Bash scripting (process management, log parsing, regex, service automation)
- Networking (DNS, DHCP, reverse proxies, VPNs, port forwarding, VLANs)
- Debugging methodology (reading logs carefully, isolating variables, testing assumptions)
- Infrastructure thinking (separation of concerns, failure modes, backup strategies)

The most valuable thing wasn't any specific technology — it was learning to debug real systems under real pressure, with real users waiting for things to work.

---

## Setup

> ⚠️ This repository documents my personal homelab. Configuration files have had sensitive values replaced with placeholders. Copy the structure, not the values.

See individual service READMEs in each subdirectory for setup instructions.

**Prerequisites:**
- Ubuntu 24.04 LTS
- Docker + Docker Compose
- Tailscale account (free tier is sufficient)

---

## Stack

`Ubuntu` `Docker` `Tailscale` `XRDP` `Jellyfin` `Immich` `Pi-hole` `Nginx Proxy Manager` `Portainer` `Uptime Kuma` `Glance` `Watchtower` `Vaultwarden` `TeamSpeak 3` `Bash` `systemd` `Python`

---

*Built by Joey (Riot) · Phase 1 of 4 · Started April 2026*
