# Riot's Arma 3 Dedicated Server

My cousins and I play Arma 3 together. For years that meant one of us had to host — which meant the server died the moment that person left, progress was inconsistent, and sessions required coordination just to get the game running. This dedicated server fixes all of that. It runs 24/7 on a home server, persists campaign progress between sessions, and manages its own headless client automatically — starting AI offloading when players connect to an active mission, and stopping it when they leave. My cousins can play whenever they want without me being online, without anyone managing a host machine, and without losing campaign progress. The server just runs.

---

## Directory Structure

```
/home/steam/arma3/
  server/            ← Arma 3 server files and mod symlinks
  configs/           ← Server config files and profiles
    home/server/
      backups/       ← Automated hourly save backups (7 days / 168 backups)
  hc/                ← Headless client profile
  steamcmd/          ← SteamCMD installation
  modsets/           ← Modset configuration files
    antistasi.txt    ← Full Antistasi modset
    impasse.txt      ← Impasse Total War modset
    current.txt      ← Tracks last used modset (auto-updated)
  start_server.sh    ← Start the server (reads current.txt for modset)
  start_hc.sh        ← Start the headless client (reads current.txt for modset)
  hc_monitor.sh      ← Auto-manages HC based on player count and mission state
  add_mod.sh         ← Script to add new mods
  remove_mod.sh      ← Script to remove mods
  add_mission.sh     ← Script to add new missions
  remove_mission.sh  ← Script to remove missions
  update_mods.sh     ← Updates all installed mods at once
  fix_lowercase.sh   ← Fix uppercase filenames in mods
  README.md          ← This file
```

---

## Systemd Services

| Service | Purpose | Auto-starts on boot |
|---|---|---|
| arma3server | Arma 3 dedicated server | Yes |
| arma3hc-monitor | HC auto-manager | Yes |
| arma3hc | Manual HC control (rarely used) | No |

---

## Server Management Commands

### Starting the Server

The server starts automatically on boot via systemd using the last used modset.

To start manually with the last used modset:

```bash
sudo systemctl start arma3server
```

To start manually with a specific modset:

```bash
sudo systemctl stop arma3server
sudo su - steam
bash /home/steam/arma3/start_server.sh antistasi
```

### Stopping the Server

```bash
sudo systemctl stop arma3server
```

### Restarting the Server

```bash
sudo systemctl restart arma3server
```

### Checking Server Status

```bash
sudo systemctl status arma3server
```

### Viewing Live Server Logs

```bash
sudo journalctl -u arma3server -f
```

Press `Ctrl+C` to stop following the log.

---

## Headless Client — Fully Automatic

The HC is managed entirely by the **HC Monitor** service. No manual intervention needed.

### How It Works

1. Monitor starts on boot alongside the server
2. Monitor reads the server log in real time
3. When a mission is loaded AND at least one real player is connected — HC starts automatically using current modset
4. When all players disconnect — HC stops automatically (mission remains marked active)
5. When a player reconnects while a mission is still loaded — HC starts again automatically
6. When the server returns to mission selection (`#missions`, vote, or mission end) — HC stops and mission is marked inactive until a new mission loads
7. HC always uses the same modset as the server via `current.txt`

### Important Notes

- HC connects as **headlessclient** — this is NOT counted as a player
- Arma 3 briefly logs a disconnect/reconnect during role selection — this is normal and does not trigger HC stop
- Antistasi Ultimate supports exactly ONE headless client
- The HC will not start until both a mission is loaded AND at least one player is connected — players sitting in the mission selection lobby do not trigger HC start

### HC Monitor Commands

```bash
# Check monitor status
sudo systemctl status arma3hc-monitor

# View monitor logs
sudo journalctl -u arma3hc-monitor -f

# Restart monitor if needed
sudo systemctl restart arma3hc-monitor
```

### Manual HC Override (if needed)

```bash
# Start HC manually with specific modset
sudo su - steam
bash /home/steam/arma3/start_hc.sh antistasi

# Stop HC manually
pkill -f "arma3server_x64.*headless"
```

### Verifying HC Connected

Look for this in server logs:

```
Player headlessclient connected (id=HC...)
```

Or in Antistasi logs:

```
Antistasi | Info | A3A_fnc_addHC | Headless Client Connected
```

---

## Modset System

Modsets allow switching between mod configurations without editing startup scripts. The server, HC, and monitor all read from `current.txt` automatically.

### Check Current Modset

```bash
cat /home/steam/arma3/modsets/current.txt
```

### List Available Modsets

```bash
ls /home/steam/arma3/modsets/*.txt | grep -v current
```

### Switch Modsets

Switching modset updates `current.txt` automatically. Both server and HC will use the new modset:

```bash
sudo systemctl stop arma3server
sudo su - steam
bash /home/steam/arma3/start_server.sh impasse
```

On next boot systemd reads `current.txt` and loads the correct modset automatically.

### Creating a New Modset

```bash
nano /home/steam/arma3/modsets/mymission.txt
```

Format — one mod per line, `#` for comments:

```
# My Mission Modset
# CBA must always be first
@cba_a3
@cup_weapons
@cup_units
@cup_vehicles
@cup_terrains_core
@cup_terrains_maps
@zeus_enhanced
```

Use it:

```bash
bash /home/steam/arma3/start_server.sh mymission
```

---

## Typical Play Session

Everything is automatic. Your cousins just connect and play.

1. Server is always running — starts on boot
2. Cousins connect and load into a mission
3. HC Monitor detects mission active + players connected — starts HC automatically
4. Session runs with HC handling AI
5. Last player disconnects — HC stops automatically
6. Cousins reconnect next day — HC starts again automatically (mission still loaded)

### Switching Modsets Before a Session

```bash
sudo systemctl stop arma3server
sudo su - steam
bash /home/steam/arma3/start_server.sh impasse
# Wait for server to fully load, then cousins connect
```

---

## Adding a New Mod

```bash
sudo su - steam
bash /home/steam/arma3/add_mod.sh <workshop_id> <mod_name>
```

Example:

```bash
bash /home/steam/arma3/add_mod.sh 450814997 cba_a3
```

After running you MUST add `@mod_name` to relevant modset files in `/home/steam/arma3/modsets/` then restart the server.

---

## Removing a Mod

```bash
sudo su - steam
bash /home/steam/arma3/remove_mod.sh <mod_name> <workshop_id>
```

Example:

```bash
bash /home/steam/arma3/remove_mod.sh cup_weapons 497660133
```

After running remove `@mod_name` from all modset files then restart the server.

---

## Adding a Mission

```bash
sudo su - steam
bash /home/steam/arma3/add_mission.sh <workshop_id> <mission_name> <map>
```

Example:

```bash
bash /home/steam/arma3/add_mission.sh 3444387114 ImpasseTotalWar Altis
```

Restart server and select mission in game as admin.

---

## Removing a Mission

```bash
sudo su - steam
bash /home/steam/arma3/remove_mission.sh ImpasseTotalWar.Altis.pbo
```

Restart server after removing.

---

## Updating All Mods

```bash
sudo systemctl stop arma3server
sudo su - steam
bash /home/steam/arma3/update_mods.sh
exit
sudo systemctl start arma3server
```

Automatically finds all installed mods by Workshop ID, updates them all in one SteamCMD session, then fixes lowercase filenames.

---

## Updating the Server

```bash
sudo systemctl stop arma3server
sudo su - steam
cd /home/steam/arma3/steamcmd
./steamcmd.sh \
  +force_install_dir /home/steam/arma3/server \
  +login YOUR_STEAM_USERNAME \
  +app_update 233780 validate \
  +quit
exit
sudo systemctl start arma3server
```

---

## Backup and Restore

### Automated Backups

Hourly backups run automatically via cron. The entire server profile directory is archived — all mission saves including Antistasi, Liberation, Impasse, and any future missions. 168 backups kept — 7 full days of hourly history.

### List Available Backups

```bash
ls -lh /home/steam/arma3/configs/home/server/backups/
```

### Restore Everything

```bash
sudo systemctl stop arma3server
sudo su - steam
tar -xzf /home/steam/arma3/configs/home/server/backups/server_saves_YYYYMMDD_HHMMSS.tar.gz \
    -C /home/steam/arma3/configs/home/server/
exit
sudo systemctl start arma3server
```

### Restore a Single File

```bash
sudo systemctl stop arma3server
sudo su - steam

# List backup contents
tar -tzf /home/steam/arma3/configs/home/server/backups/server_saves_YYYYMMDD_HHMMSS.tar.gz

# Extract specific file
tar -xzf /home/steam/arma3/configs/home/server/backups/server_saves_YYYYMMDD_HHMMSS.tar.gz \
    -C /home/steam/arma3/configs/home/server/ \
    ./AntistasiUltimate.vars

exit
sudo systemctl start arma3server
```

### Manual Backup

```bash
sudo su - steam
mkdir -p /home/steam/arma3/configs/home/server/backups
tar -czf /home/steam/arma3/configs/home/server/backups/server_saves_manual_$(date +%Y%m%d_%H%M%S).tar.gz \
    -C /home/steam/arma3/configs/home/server \
    --exclude=backups .
```

---

## In-Game Admin

```
#login YOUR_ADMIN_PASSWORD
```

---

## Save File Locations

| Mission | Save File |
|---|---|
| Antistasi Ultimate | /home/steam/arma3/configs/home/server/AntistasiUltimate.vars |
| Liberation RX | Created automatically on first play |
| Impasse Total War | Created automatically on first play |

All save files backed up automatically every hour.

---

## Monitoring — Uptime Kuma

The Arma 3 server is monitored via Uptime Kuma using a **Push monitor** with a cron job heartbeat. This approach was chosen after two other monitor types failed:

- **Steam Game Server monitor** — requires SteamAPI which the dedicated server intentionally runs without. Returns `Steam API Key not found` and never goes green.
- **TCP Port monitor on 2302** — fails because Arma 3 uses UDP not TCP. Returns `ECONNREFUSED` even when the server is running.

The Push monitor is the most accurate option — the server actively reports its own health rather than being passively queried.

### How It Works

A cron job runs every minute as the riot user. It checks whether the `arma3server` systemd service is active and sends a heartbeat to Uptime Kuma only if it is. If the service stops the heartbeats stop and Uptime Kuma alerts via Discord after the configured retry window.

### Cron Job Setup

```bash
# As riot user
crontab -e
```

Add:

```
* * * * * systemctl is-active --quiet arma3server && curl -s "http://YOUR_SERVER_IP:3001/api/push/YOUR_PUSH_TOKEN" > /dev/null 2>&1
```

Replace `YOUR_PUSH_TOKEN` with the token from your Uptime Kuma Push monitor URL.

### Important — Use IP Address Not DNS Name

Server-side scripts must use the direct IP address rather than local DNS names like `uptime-kuma.riot-homelab`. This is a split-horizon DNS issue — the server can't reliably resolve its own domain names because it's asking Pi-hole (which runs on the same machine) to resolve an address that points back to itself. Always use `YOUR_SERVER_IP:3001` directly in cron jobs and scripts running on the homelab server.

### Uptime Kuma Monitor Settings

- **Type:** Push
- **Friendly Name:** Arma 3 Server
- **Heartbeat Interval:** 60 seconds
- **Retries:** 3 (alerts after 3 missed heartbeats — ~3 minutes)

### Verifying the Cron Job is Running

```bash
# Check cron is sending heartbeats
grep CRON /var/log/syslog | tail -10

# Manually trigger the heartbeat to test
curl -s "http://YOUR_SERVER_IP:3001/api/push/YOUR_PUSH_TOKEN"
```

---

## HC Monitor — Technical Notes

The monitor tracks two independent state variables and requires both conditions to be true before starting the HC:

- **`MISSION_ACTIVE`** — true only after a mission has fully initialized; false in lobby or before any mission loads
- **`PLAYER_COUNT`** — net count of real players currently connected

The HC runs if and only if `MISSION_ACTIVE=true` AND `PLAYER_COUNT > 0`.

| Situation | MISSION_ACTIVE | PLAYER_COUNT | HC |
|---|---|---|---|
| Mission running, players on | true | ≥1 | Running |
| All players log off for the night | true | 0 | Stopped |
| Player reconnects, mission still loaded | true | 1 | Started |
| Admin runs `#missions` or mission ends | false | ≥1 | Stopped |
| Server restart | false | 0 | Stopped |

**Regex pattern `[^dis]connected`** — matches `connected` but NOT `disconnected`. This prevents disconnect events from being double-counted as connections.

**Pipe character filter `grep -v "|"`** — all Arma 3 server events use simple format. All Antistasi internal logs contain pipe characters. This cleanly separates real server events from mission script logs.

**Headless client filter `grep -vi "headless"`** — HC connects as `headlessclient` which contains `headless`. This prevents the HC from counting as a real player.

**Baseline on startup** — on startup the monitor reads the log from the last `Game Port: 2302` line forward and calculates both current player count and mission state. This means the monitor correctly handles players and missions that were already active before it started.

**Mission state tracking** — the monitor watches for the CBA `MISSIONINIT:` log line that appears for every mission when it finishes initializing. The HC will not start until this line is seen, preventing premature HC starts while players sit in the mission selection lobby.

**Lobby-return detection** — the monitor watches for `Waiting for next game.` in the server log, which fires whenever the server returns to mission selection (admin `#missions` command, end-of-mission vote, or natural mission completion). This stops the HC and clears mission state immediately, even when players stay connected through the transition. A server restart (`Game Port: 2302`) also clears all state.

**Mission remains active with zero players** — when all players disconnect for the night, `MISSION_ACTIVE` stays true. The HC stops because the player condition fails, but the moment anyone reconnects the HC starts again immediately without waiting for a new `MISSIONINIT:` line. Only `Waiting for next game.` or a server restart clears mission state.

**Modset awareness** — the `start_hc` function reads `current.txt` at the moment it starts the HC. This means if you switch modsets the HC always loads the correct one.

---

## Verifying HC Monitor Works

Use this diagnostic to test monitor logic without starting it:

```bash
sudo su - steam
bash -c '
LOG="/home/steam/arma3/server/server_log.txt"

LAST_START=$(grep -n "Game Port: 2302" "$LOG" | tail -1 | cut -d: -f1)
echo "Server started at log line: $LAST_START"
echo ""

echo "=== CONNECTIONS ==="
tail -n +"$LAST_START" "$LOG" | grep -E "Player.*[^dis]connected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|"
echo ""

echo "=== DISCONNECTIONS ==="
tail -n +"$LAST_START" "$LOG" | grep "Player.*disconnected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|"
echo ""

echo "=== COUNTS ==="
CONNECTED=$(tail -n +"$LAST_START" "$LOG" | grep -E "Player.*[^dis]connected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|" | wc -l)
DISCONNECTED=$(tail -n +"$LAST_START" "$LOG" | grep "Player.*disconnected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|" | wc -l)
echo "Connected: $CONNECTED"
echo "Disconnected: $DISCONNECTED"
echo "Net player count: $((CONNECTED - DISCONNECTED))"
echo ""

echo "=== MISSION STATE ==="
LOG_SLICE=$(tail -n +"$LAST_START" "$LOG")
LAST_MISSIONINIT_LINE=$(echo "$LOG_SLICE" | grep -n "MISSIONINIT:"          | tail -1 | cut -d: -f1)
LAST_LOBBY_LINE=$(      echo "$LOG_SLICE" | grep -n "Waiting for next game" | tail -1 | cut -d: -f1)
LAST_MISSIONINIT_LINE=${LAST_MISSIONINIT_LINE:-0}
LAST_LOBBY_LINE=${LAST_LOBBY_LINE:-0}

if [ "$LAST_MISSIONINIT_LINE" -gt "$LAST_LOBBY_LINE" ]; then
    MISSION_NAME=$(echo "$LOG_SLICE" | grep "MISSIONINIT:" | tail -1 | grep -oP "missionName=\K[^,]+")
    echo "Mission active: YES ($MISSION_NAME)"
else
    echo "Mission active: NO (in lobby or pre-mission)"
fi

if [ "$LAST_LOBBY_LINE" -gt 0 ]; then
    echo "Last lobby return at log slice line: $LAST_LOBBY_LINE"
else
    echo "Last lobby return: none since server start"
fi
echo ""

echo "=== HC SHOULD BE RUNNING? ==="
NET=$((CONNECTED - DISCONNECTED))
[ "$NET" -lt 0 ] && NET=0
if [ "$LAST_MISSIONINIT_LINE" -gt "$LAST_LOBBY_LINE" ] && [ "$NET" -gt 0 ]; then
    echo "YES — mission active and $NET player(s) connected"
else
    echo "NO  — $([ "$LAST_MISSIONINIT_LINE" -le "$LAST_LOBBY_LINE" ] && echo "no active mission" || echo "no players connected")"
fi
echo ""

echo "=== HC ACTUALLY RUNNING? ==="
if pgrep -f "arma3server_x64.*headless" > /dev/null 2>&1; then
    echo "YES — PID $(pgrep -f "arma3server_x64.*headless")"
else
    echo "NO"
fi
'
```

Expected results with no players and no mission: mission active NO, HC should be running NO, HC actually running NO.

---

## Current Mod List

| Mod Name | Workshop ID | Symlink |
|---|---|---|
| CBA_A3 | 450814997 | @cba_a3 |
| CUP Weapons | 497660133 | @cup_weapons |
| CUP Units | 497661914 | @cup_units |
| CUP Vehicles | 541888371 | @cup_vehicles |
| CUP Terrains Core | 583496184 | @cup_terrains_core |
| CUP Terrains Maps | 583544987 | @cup_terrains_maps |
| Antistasi Ultimate | 3020755032 | @antistasi |
| Enhanced Movement | 333310405 | @enhanced_movement |
| Enhanced Movement Rework | 2034363662 | @enhanced_movement_rework |
| Remove Stamina | 632435682 | @remove_stamina |
| Zeus Enhanced | 1779063631 | @zeus_enhanced |
| DUI Squad Radar | 1638341685 | @dui_squad_radar |
| Better Inventory | 2791403093 | @better_inventory |
| Arsenal Search | 2060770170 | @arsenal_search |
| Reload While Aiming | 3450227250 | @reload_while_aiming |
| Dismount Where You Look | 1841553455 | @dismount_where_you_look |
| Swim Faster | 1808723766 | @swim_faster |
| AI Cannot See Drones | 2947745583 | @ai_no_see_drones |
| AI Heli Decelerate | 3496308284 | @ai_heli_decelerate |
| AGC Garbage Collector | 1724884525 | @agc |
| Auto ViewDistance | 1544955993 | @auto_viewdistance |
| Hide Among Grass | 3346427969 | @hide_among_grass |
| Additional Measurements | 1942567517 | @additional_measurements |
| L3-GPNVG18 | 313041182 | @l3_gpnvg18 |

---

## Current Modsets

### antistasi.txt
Full mod list including @antistasi framework.

### impasse.txt
Same as antistasi but without @antistasi mod.

---

## Current Missions

| Mission | Map | File |
|---|---|---|
| Antistasi Ultimate | Altis | Built into @antistasi mod |
| Liberation RX | Altis | LiberationRX.Altis.pbo |
| Impasse Total War | Altis | ImpasseTotalWar.Altis.pbo |

---

## Port Forwarding Reference

| Port | Protocol | Purpose |
|---|---|---|
| 2302 | UDP | Main game traffic |
| 2303 | UDP | Steam query port |
| 2304 | UDP | BattlEye |
| 2305 | UDP | Additional game port |
| 2306 | UDP | VON voice |
| 2344 | UDP | Headless client |
| 2345 | UDP | Headless client |
| 27016 | UDP | Steam server browser |

---

## Discord Bot (Planned)

A Discord bot is planned for cousin server management. Built with Python and discord.py, runs as a Docker container.

| Command | Action |
|---|---|
| !status | Server status, player count, current modset |
| !modset list | List available modsets |
| !modset \<name\> | Switch modset and restart |
| !restart | Restart the server |
| !backup | Trigger manual backup |

---

## Troubleshooting

### Server Won't Start

```bash
sudo journalctl -u arma3server -f
cat /home/steam/arma3/server/server_log.txt | tail -50
```

### HC Not Starting When Players Connect

```bash
sudo systemctl status arma3hc-monitor
sudo journalctl -u arma3hc-monitor -f
sudo systemctl restart arma3hc-monitor
```

### HC Player Count Wrong

Clear the log and restart everything for a clean slate:

```bash
sudo systemctl stop arma3hc-monitor
sudo systemctl stop arma3server
sudo su - steam
> /home/steam/arma3/server/server_log.txt
exit
sudo systemctl start arma3server
# Wait for mission to load, then start monitor
sudo systemctl start arma3hc-monitor
```

### HC Wrong Password

- Verify server join password matches in `start_hc.sh` and `server.cfg`
- Make sure `-headless` flag is present in `start_hc.sh`
- Single quote passwords containing special characters

### Mod Uppercase Filename Errors

```bash
sudo su - steam
bash /home/steam/arma3/fix_lowercase.sh
```

### Mission Not Showing In Game

- Verify `.pbo` in `/home/steam/arma3/server/mpmissions/`
- Check filename format: `MissionName.Map.pbo`

### Antistasi Save Not Loading

- Verify `AntistasiUltimate.vars` is in `/home/steam/arma3/configs/home/server/`
- Restore from backup if corrupted

### Wrong Modset on Boot

```bash
cat /home/steam/arma3/modsets/current.txt
echo "antistasi" > /home/steam/arma3/modsets/current.txt
```

### Uptime Kuma Push Monitor Not Receiving Heartbeats

- Verify cron job is in riot user's crontab: `crontab -l`
- Verify the push URL uses IP address not DNS name — `http://YOUR_SERVER_IP:3001/api/push/YOUR_TOKEN`
- Test manually: `curl -s "http://YOUR_SERVER_IP:3001/api/push/YOUR_TOKEN"`
- Check arma3server is actually running: `systemctl is-active arma3server`

### Kill All Arma Processes

```bash
sudo su - steam
pkill -f arma3server_x64
```
