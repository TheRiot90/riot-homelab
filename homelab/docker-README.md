# Homelab — Docker Services

All services run as Docker containers managed with Docker Compose. Each service has its own subdirectory containing a `docker-compose.yml` and where applicable an `.env.example` file.

---

## Quick Reference

| Service | URL | Port | Purpose |
|---|---|---|---|
| Jellyfin | jellyfin.riot-homelab | 8096 | Media server |
| Immich | immich.riot-homelab | 2283 | Photo & video backup |
| Pi-hole | pihole.riot-homelab/admin | 8080 | DNS ad blocker + local DNS |
| Nginx Proxy Manager | npm.riot-homelab | 81 | Reverse proxy + SSL |
| Portainer | portainer.riot-homelab | 9000 | Docker management UI |
| Uptime Kuma | 192.168.1.200:3001 | 3001 | Service monitoring |
| Glance | glance.riot-homelab | 8585 | Homelab dashboard |
| Watchtower | — | — | Automatic container updates |
| Vaultwarden | vaultwarden.riot-homelab | 8181 | Self-hosted password manager |
| TeamSpeak 3 | YOUR_SERVER_IP:9987 | 9987 UDP | Voice comms (TFAR support) |

---

## Prerequisites

- Ubuntu 24.04 LTS
- Docker Engine + Docker Compose plugin
- Tailscale installed and authenticated
- Pi-hole running and configured as local DNS

### Install Docker

```bash
sudo apt update
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker $USER
```

Log out and back in after adding yourself to the docker group.

---

## Folder Structure

```
homelab/
  jellyfin/
    docker-compose.yml
    config/          ← Jellyfin config (not committed)
    movies/          ← Media library (not committed)
    tv/              ← TV library (not committed)
  immich/
    docker-compose.yml
    .env.example
    config/          ← Immich config (not committed)
  pihole/
    docker-compose.yml
    config/          ← Pi-hole config (not committed)
    dnsmasq/         ← DNS config (not committed)
  nginx-proxy-manager/
    docker-compose.yml
    data/            ← NPM data (not committed)
    letsencrypt/     ← SSL certs (not committed)
  portainer/
    docker-compose.yml
    data/            ← Portainer data (not committed)
  uptime-kuma/
    docker-compose.yml
    data/            ← Kuma data (not committed)
  glance/
    docker-compose.yml
    glance.yml       ← Dashboard config
  watchtower/
    docker-compose.yml
  vaultwarden/
    docker-compose.yml
    data/            ← Vault data (not committed)
  teamspeak/
    docker-compose.yml
    data/            ← TS3 data (not committed)
```

---

## Services

### Jellyfin

Self-hosted media server. Serves movies and TV shows to any device on the network or remotely via Tailscale.

**Key configuration:**
- Uses Intel Quick Sync for hardware accelerated transcoding — no GPU required
- Media libraries mapped as Docker volumes from host filesystem
- Accessible via Jellyfin apps on iOS, Android, smart TVs, and browsers

**Start:**
```bash
cd jellyfin
docker compose up -d
```

**Notes:**
- Point media libraries at `/media/movies` and `/media/tv` inside the container
- These map to your actual media folders on the host via volumes in `docker-compose.yml`
- Add new media libraries through the Jellyfin web UI at first launch

---

### Immich

Self-hosted Google Photos alternative. Handles automatic photo and video backup from mobile devices, face recognition, smart search, and album management.

**Key configuration:**
- Requires a `.env` file — copy `.env.example` and fill in values
- Photo data stored outside the container via volume mount
- Machine learning disabled to reduce resource usage (photos still back up fully)
- Four containers: `immich_server`, `immich_machine_learning`, `immich_redis`, `immich_postgres`

**Start:**
```bash
cd immich
cp .env.example .env
# Edit .env with your values
docker compose up -d
```

**Storage note:** Photo libraries grow large quickly — especially with video. Plan storage accordingly before enabling backups for multiple users. Point the data volume at a drive with sufficient capacity.

---

### Pi-hole

Network-wide DNS ad blocker and local DNS server. Blocks ads at the DNS level for every device on the network. Also handles local DNS records so services are accessible by friendly name rather than IP:port.

**Key configuration:**
- Runs with `network_mode: host` — requires direct network access to handle DNS on port 53
- Web interface on port 8080 (port 80 conflicts with Nginx Proxy Manager)
- `systemd-resolved` must be disabled on the host to free port 53

**Disable systemd-resolved before starting:**
```bash
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

**Start:**
```bash
cd pihole
docker compose up -d
```

**After starting — set admin password:**
```bash
docker exec -it pihole pihole setpassword
```

**Local DNS records to add via Pi-hole dashboard:**

Go to **Settings → Local DNS → DNS Records** and add:

| Domain | IP |
|---|---|
| riot-homelab | YOUR_SERVER_IP |
| jellyfin.riot-homelab | YOUR_SERVER_IP |
| immich.riot-homelab | YOUR_SERVER_IP |
| portainer.riot-homelab | YOUR_SERVER_IP |
| npm.riot-homelab | YOUR_SERVER_IP |
| pihole.riot-homelab | YOUR_SERVER_IP |
| glance.riot-homelab | YOUR_SERVER_IP |
| uptime-kuma.riot-homelab | YOUR_SERVER_IP |
| vaultwarden.riot-homelab | YOUR_SERVER_IP |

**Point devices at Pi-hole for DNS:**
Set DNS to `YOUR_SERVER_IP` on each device. Use `1.1.1.1` as secondary DNS for fallback when the server is offline.

---

### Nginx Proxy Manager

Reverse proxy that routes traffic to services by domain name, eliminating the need to remember port numbers. Manages SSL certificates.

**Key configuration:**
- Listens on ports 80 (HTTP), 443 (HTTPS), and 81 (admin UI)
- Port 80 must be free — conflicts with Pi-hole's default web port (resolved by running Pi-hole on 8080)

**Start:**
```bash
cd nginx-proxy-manager
docker compose up -d
```

**Default login (change immediately):**
- Email: `admin@example.com`
- Password: `changeme`

**Add proxy hosts for each service:**

| Domain | Forward Host | Forward Port |
|---|---|---|
| jellyfin.riot-homelab | YOUR_SERVER_IP | 8096 |
| immich.riot-homelab | YOUR_SERVER_IP | 2283 |
| portainer.riot-homelab | YOUR_SERVER_IP | 9000 |
| pihole.riot-homelab | YOUR_SERVER_IP | 8080 |
| npm.riot-homelab | YOUR_SERVER_IP | 81 |
| glance.riot-homelab | YOUR_SERVER_IP | 8585 |
| vaultwarden.riot-homelab | YOUR_SERVER_IP | 8181 |

Enable **Block Common Exploits** and **Websockets Support** on each proxy host.

---

### Portainer

Web UI for Docker management. View container status, logs, resource usage, and control containers without touching the terminal.

**Start:**
```bash
cd portainer
docker compose up -d
```

**Access:** `http://YOUR_SERVER_IP:9000`

Create admin account on first launch. Select **Docker Standalone** environment pointing at the local Docker socket.

---

### Uptime Kuma

Service monitoring with Discord notifications. Monitors all homelab services and alerts when something goes down or comes back up.

**Start:**
```bash
cd uptime-kuma
docker compose up -d
```

**Monitors to configure:**

| Name | Type | URL / Host | Port |
|---|---|---|---|
| Jellyfin | HTTP | http://YOUR_SERVER_IP:8096 | — |
| Immich | HTTP | http://YOUR_SERVER_IP:2283 | — |
| Pi-hole | HTTP | http://YOUR_SERVER_IP:8080/admin | — |
| Nginx Proxy Manager | HTTP | http://YOUR_SERVER_IP:81 | — |
| Portainer | HTTP | http://YOUR_SERVER_IP:9000 | — |
| Uptime Kuma | HTTP | http://YOUR_SERVER_IP:3001 | — |
| Glance | HTTP | http://YOUR_SERVER_IP:8585 | — |
| Vaultwarden | HTTP | http://YOUR_SERVER_IP:8181 | — |
| TeamSpeak | TCP | YOUR_SERVER_IP | 9987 |
| Pi-hole DNS | DNS | google.com (resolver: YOUR_SERVER_IP) | 53 |

**Discord notifications:**
1. Create a Discord webhook in your server under a dedicated channel (e.g. `#homelab-alerts`)
2. In Uptime Kuma go to **Settings → Notifications → Setup Notification → Discord**
3. Paste the webhook URL and test
4. Apply the notification to all monitors

---

### Glance

Lightweight homelab dashboard. Shows service status, system stats, weather, RSS feeds, and bookmarks to all services in one page.

**Start:**
```bash
cd glance
docker compose up -d
```

**Configuration:** Edit `glance.yml` to customize widgets, feeds, and bookmarks. Use direct IP:port URLs for service monitors rather than proxy names to avoid false errors when Nginx is the issue.

**Note:** Port mapping is `8585:8080` — Glance runs internally on 8080 but is exposed on 8585 to avoid conflict with Pi-hole.

---

### Watchtower

Automatically checks for and applies Docker image updates. Runs at 4AM daily, removes old images after updating, and logs what was updated.

**Start:**
```bash
cd watchtower
docker compose up -d
```

No configuration needed after initial start. Monitors all running containers automatically — no setup required when adding new services.

**To exclude a container from auto-updates** add this label to its compose file:
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```

---

### Vaultwarden

Self-hosted Bitwarden-compatible password manager. Uses the official Bitwarden mobile and browser extension clients.

**⚠️ Requires HTTPS to function** — the browser's Subtle Crypto API used for encryption only works in a secure context. Set up a real SSL certificate via DeSEC.io and Let's Encrypt before attempting to use Vaultwarden.

**Key configuration:**
- `DOMAIN` must match your actual HTTPS URL
- `SIGNUPS_ALLOWED` — set to `true` during initial account creation, then `false` after
- `ADMIN_TOKEN` — use an Argon2 hashed token, not plain text

**Hashing the admin token:**
```bash
docker exec -it vaultwarden /vaultwarden hash
```

**Start:**
```bash
cd vaultwarden
docker compose up -d
```

**After creating accounts** disable signups:
```yaml
SIGNUPS_ALLOWED: false
```
Then restart: `docker compose down && docker compose up -d`

**Admin panel:** `http://YOUR_SERVER_IP:8181/admin`

---

### TeamSpeak 3

Voice communications server. Required for the TFAR (Task Force Arrowhead Radio) mod used with the Arma 3 server.

**Key configuration:**
- Free license supports up to 32 slots — sufficient for any home group
- Only UDP port 9987 is strictly required for voice
- TCP ports 10011 and 30033 are optional (ServerQuery and file transfer)

**Start:**
```bash
cd teamspeak
docker compose up -d
```

**Get the admin token immediately after first start:**
```bash
docker logs teamspeak 2>&1 | grep token
```

This token only appears once. Use it to claim server admin rights when connecting for the first time via the TeamSpeak client.

**Connect:** `YOUR_SERVER_IP:9987`

---

## Common Operations

### Start All Services

```bash
for dir in jellyfin immich pihole nginx-proxy-manager portainer uptime-kuma glance watchtower vaultwarden teamspeak; do
    echo "Starting $dir..."
    cd /home/riot/homelab/$dir && docker compose up -d
    cd /home/riot/homelab
done
```

### Stop All Services

```bash
for dir in jellyfin immich pihole nginx-proxy-manager portainer uptime-kuma glance watchtower vaultwarden teamspeak; do
    echo "Stopping $dir..."
    cd /home/riot/homelab/$dir && docker compose down
    cd /home/riot/homelab
done
```

### Check All Container Status

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### View Logs for a Specific Container

```bash
docker logs <container_name> --tail 50 -f
```

### Update a Specific Service Manually

```bash
cd /home/riot/homelab/<service>
docker compose pull
docker compose down
docker compose up -d
```

### Check Disk Usage by Service

```bash
du -sh /home/riot/homelab/*
```

---

## Networking Architecture

```
Device request for jellyfin.riot-homelab
  │
  ├── Pi-hole DNS resolves → YOUR_SERVER_IP
  │
  └── Nginx Proxy Manager (port 80/443)
        └── Routes to Jellyfin (port 8096)
```

All services communicate on Docker's internal network. Only necessary ports are exposed to the host.

Remote access via Tailscale — devices connected to the Tailscale network reach services using the server's Tailscale IP instead of the local IP. No port forwarding required.

---

## Troubleshooting

### Container Won't Start

```bash
docker logs <container_name>
```

### Port Already in Use

```bash
sudo ss -tlnp | grep <port_number>
```

### Pi-hole Not Resolving DNS

```bash
# Verify Pi-hole is running
docker ps | grep pihole

# Test DNS resolution
nslookup google.com YOUR_SERVER_IP

# Check port 53 is free
sudo ss -tlnp | grep 53
```

### Nginx Proxy Manager 502 Bad Gateway

- Verify the target container is running
- Check the forwarded port is correct
- Confirm the container is on a network Nginx can reach

### Immich Not Receiving Uploads

- Verify all four Immich containers are running (server, machine_learning, redis, postgres)
- Check storage volume has sufficient space: `df -h`
- Verify `.env` values are correct

### Reset a Service Completely

```bash
cd /home/riot/homelab/<service>
docker compose down -v    # -v removes volumes — destructive, loses data
docker compose up -d
```

> ⚠️ The `-v` flag deletes all data volumes. Only use this if you intend to start completely fresh.
