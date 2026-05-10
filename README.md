# Riot Homelab

A self-hosted home lab built from the ground up on a borrowed PC — running a media server, photo backup system, network-wide ad blocking, remote access, service monitoring, and a fully automated Arma 3 dedicated game server with headless client management.

This started as a personal project to stop paying for streaming subscriptions and to give my cousins a persistent game server they could access without me being online. It grew into something I'm genuinely proud of — a real infrastructure project that taught me Linux, Docker, networking, bash scripting, and systems thinking in a way that no tutorial ever could.

---

## Why I Built This

I had a few specific problems:

- **Media** — paying for multiple streaming services for content I already owned on DVD or could source myself. Wanted a self-hosted Plex/Jellyfin alternative that I fully controlled.
- **Photos** — my girlfriend had 42,000 photos and videos on her phone with no backup strategy. Losing those was not acceptable. Didn't want to hand her memories to Google or Apple.
- **Gaming** — my cousins and I play Arma 3 together. Every session required one of us to host, meaning the server died when that person left. Wanted a persistent dedicated server our group could access any time, even when I wasn't available.
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

### Arma 3 Dedicated Server
Running natively (not in Docker) as a dedicated `steam` system user, managed by systemd. Features I built from scratch:

- **Modset system** — switch between mod configurations via text files, persisted across reboots
- **Headless client auto-management** — a bash monitor script that follows the server log in real time, automatically starts the AI headless client when players connect to an active mission, and stops it when the last player leaves
- **Automated backups** — hourly cron job creates compressed archives of all campaign save data, retaining 7 days of history
- **Mod management scripts** — add, remove, and update mods via simple bash scripts

---

## Architecture

See [`architecture.html`](./architecture.html) for a visual diagram of the full stack.

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
              └── Arma 3 Server (systemd)
                    ├── arma3server_x64 :2302
                    ├── Headless Client (auto-managed)
                    └── HC Monitor (hc_monitor.sh)

Remote Access: Tailscale mesh VPN (all devices)
Local DNS: Pi-hole (*.riot-homelab → 192.168.1.200)
```

---

## Interesting Technical Challenges

A few problems that weren't solved by copy-pasting a tutorial:

**HC Monitor player counting** — Arma 3 server logs are noisy. The word "disconnected" contains "connected", so naive grep would double-count every disconnect. Antistasi mission scripts emit their own "Player disconnected" log lines that look identical to real server events but aren't. The solution was combining a regex negative lookbehind `[^dis]connected` with a pipe character filter `grep -v "|"` to cleanly separate real server events from mission script logs.

**Arma 3 on Linux** — Workshop mods downloaded via SteamCMD are not guaranteed to have lowercase filenames. Linux is case-sensitive. Arma 3 on Linux requires all mod files to be lowercase or it refuses to load them. Built a script that recursively lowercases every filename in every mod directory.

**Immich storage overflow** — 42,000 photos and videos exceeded available SSD space mid-sync. Solved by reformatting a spare 1TB external drive as ext4, mounting it persistently via fstab with the `nofail` flag, and live-migrating Immich's data directory by updating the Docker Compose volume mapping and restarting the container without data loss.

**SteamCMD install location inconsistency** — `force_install_dir` in SteamCMD reliably sets the install path for `app_update` commands but inconsistently for `workshop_download_item` commands, causing mods to split across two directories. Solved by consolidating all workshop content post-download and building a modset system that references content by symlink rather than absolute path.

---

## Roadmap

- [ ] DeSEC.io domain + Let's Encrypt SSL for proper HTTPS
- [ ] Vaultwarden fully operational (requires HTTPS)
- [ ] SSH hardening (key-based auth only)
- [ ] File sharing solution (Filebrowser or Samba)
- [ ] Discord bot in Python for cousin server management
- [ ] GitHub Actions for automated deployment documentation
- [ ] Phase 2: UCG-Max firewall, proper network segmentation, VLANs
- [ ] Phase 3: Dedicated Unraid NAS, migrate all services, Arma 3 to Ubuntu VM

---

## What I Learned

I came into this knowing Python and some programming fundamentals. I left with practical experience in:

- Linux system administration (Ubuntu, systemd, cron, file permissions, users)
- Docker and Docker Compose (containers, volumes, networks, multi-service stacks)
- Bash scripting (process management, log parsing, regex, service automation)
- Networking (DNS, DHCP, reverse proxies, VPNs, port forwarding, VLANs)
- Debugging methodology (reading logs carefully, isolating variables, testing assumptions)
- Infrastructure thinking (separation of concerns, failure modes, backup strategies)

The most valuable thing wasn't any specific technology — it was learning to debug real systems under real pressure, with real users (my cousins) waiting for things to work.

---

## Setup

> ⚠️ This repository documents my personal homelab. Configuration files have had sensitive values replaced with placeholders. Copy the structure, not the values.

See individual service READMEs in each subdirectory for setup instructions.

**Prerequisites:**
- Ubuntu 24.04 LTS
- Docker + Docker Compose
- Tailscale account (free tier is sufficient)
- Steam account (for Arma 3 server download)

---

## Stack

`Ubuntu` `Docker` `Tailscale` `XRDP` `Jellyfin` `Immich` `Pi-hole` `Nginx Proxy Manager` `Portainer` `Uptime Kuma` `Glance` `Watchtower` `Vaultwarden` `TeamSpeak 3` `Bash` `systemd` `Python (planned)`

---

*Built by Joey (Riot) · Phase 1 of 4 · Started May 2026*
