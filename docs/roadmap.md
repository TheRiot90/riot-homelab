# Riot Homelab — Master Roadmap

## Current Hardware
- **Server:** Girlfriend's PC (i5-12400F, 32GB RAM, 1TB SSD + 1TB external USB)
- **Networking:** Altice ISP gateway (locked down)
- **Storage:** 1TB external USB (temporary Immich overflow)

## Future Hardware (Planned)
- Dedicated NAS box running Unraid (4-6 months)
- Own modem (cable, DOCSIS 3.1, Optimum approved)
- Ubiquiti Cloud Gateway Max
- UniFi Access Point
- Additional NAS drives (4-5x to fill bays)

---

## Phase 1 — Foundation (Current Phase)
**Goal:** Get core services running on girlfriend's PC. Learn the stack.

### Completed ✅
- Ubuntu 24.04 LTS dual boot on girlfriend's PC
- Static IP configured
- XRDP remote desktop from Windows/Linux laptop
- Docker installed and configured
- Tailscale VPN for remote access
- Jellyfin media server (Cosby Show S1 loaded)
- Immich photo backup (girlfriend's 42k photos syncing)
- Pi-hole DNS ad blocking + local DNS records
- Portainer Docker management UI
- Nginx Proxy Manager reverse proxy
- Watchtower automatic container updates
- Uptime Kuma service monitoring
- Glance dashboard
- TeamSpeak 3 server (for TFAR mod)
- Arma 3 dedicated server with:
  - Full mod list including CUP suite
  - Antistasi, Liberation RX, Impasse Total War missions
  - Headless client with auto-start/stop monitor
  - Modset switching system
  - Automated hourly backups (7 days retention)
  - Systemd services for auto-start on boot
  - Campaign save working (AntistasiUltimate.vars)

### In Progress 🔄
- Vaultwarden password manager (blocked on HTTPS)
- DeSEC.io domain + Let's Encrypt SSL certificates
- File sharing solution (GUI based)

### Pending 📋
- SSH hardening (key-based auth, non-standard port)
- BattlEye enable on Arma 3 server
- TFAR mod added to Arma 3 server
- Discord bot for cousin server management (Python)
- Arma 3 server campaign save automated backup verification
- Glance widget fixes (Pi-hole and NPM showing errors)
- TeamSpeak added to Uptime Kuma and Glance
- Immich machine learning disabled cleanly
- Vaultwarden fully operational with HTTPS
- Password audit — replace reused passwords with Vaultwarden generated ones
- GitHub repository for entire project

---

## Phase 2 — Networking Upgrade
**Goal:** Proper firewall, clean network segmentation, reliable remote access.

### Hardware to Buy
- Own cable modem (DOCSIS 3.1, Optimum approved, ~$100)
- Ubiquiti Cloud Gateway Max ($199)
- UniFi Access Point U6 Lite (~$99)

### Tasks
- Replace Altice gateway with own modem
- Install and configure UCG-Max
- Set up proper VLANs:
  - Main network (PCs, phones, server)
  - IoT devices
  - Game server
  - Guest network
- Configure UCG-Max as DHCP server (Pi-hole gets proper network-wide DNS)
- Set up WireGuard VPN on UCG-Max as Tailscale companion
- Configure UniFi AP for WiFi
- Set up IDS/IPS on UCG-Max
- DeSEC.io domain pointing to Tailscale IP for proper SSL
- Port forwarding cleaned up and documented

---

## Phase 3 — NAS Build
**Goal:** Dedicated always-on hardware. Free girlfriend's PC permanently.

### Hardware to Buy
- NAS compute hardware (TBD based on Phase 1 learnings)
- 4-5x HDDs (targeting 10-20TB usable in RAIDZ1)
- UPS battery backup

### Software
- Unraid OS ($69 one time license)
- Migrate all Docker services from girlfriend's PC to Unraid
- Arma 3 server migrated to Ubuntu VM inside Unraid
- Vaultwarden becomes primary password manager (always-on hardware)
- TrueNAS or Unraid handles storage with ZFS/parity protection

### Migration Plan
1. Build Unraid box
2. Set up all Docker services on Unraid
3. Test everything works
4. Migrate Immich data from external USB to NAS drives
5. Migrate Jellyfin media library
6. Set up Arma 3 Ubuntu VM
7. Migrate Arma 3 server to VM
8. Remove Ubuntu from girlfriend's PC
9. Return PC to girlfriend — clean slate

---

## Phase 4 — Home Security
**Goal:** Camera system integrated with homelab.

### Hardware to Buy
- Upgrade UCG-Max to Cloud Gateway Max (if not already done in Phase 2)
- UniFi cameras (2-3x to start, G4 Instant or similar)
- PoE switch if needed

### Tasks
- Set up UniFi Protect on UCG-Max
- Camera VLAN — isolated from main network
- Motion detection and recording to NAS
- Remote viewing through Tailscale

---

## Future Ideas (No Timeline)
- 7 Days to Die server (Ubuntu VM on Unraid)
- Additional game servers as needed (all as VMs)
- Raspberry Pi as always-on emergency access device
- Home automation integration
- Grafana + Prometheus for advanced monitoring
- Self hosted Git server (Gitea)
- Self hosted note taking (Obsidian sync or Joplin)
- Nextcloud for full Google Drive replacement

---

## Tech Stack Reference

| Category | Technology |
|---|---|
| Server OS | Ubuntu 24.04 LTS |
| Container runtime | Docker + Docker Compose |
| Container management | Portainer |
| NAS OS (future) | Unraid |
| Remote access | Tailscale (WireGuard) |
| Remote desktop | XRDP + XFCE |
| DNS | Pi-hole |
| Reverse proxy | Nginx Proxy Manager |
| SSL (planned) | Let's Encrypt via DeSEC.io |
| Monitoring | Uptime Kuma |
| Dashboard | Glance |
| Updates | Watchtower |
| Password manager | Vaultwarden (Bitwarden compatible) |
| Media server | Jellyfin |
| Photo backup | Immich |
| Voice comms | TeamSpeak 3 |
| Game server OS | Ubuntu (bare metal now, VM on Unraid later) |
| Game server mgmt | Custom bash scripts + systemd |
| Bot language | Python + discord.py (planned) |
| Version control | GitHub |
