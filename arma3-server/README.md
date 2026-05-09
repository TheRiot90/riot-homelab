# Riot's Arma 3 Dedicated Server

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
  hc_monitor.sh      ← Auto-manages HC based on player count
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
3. When first real player connects — HC starts automatically using current modset
4. When last real player disconnects — HC stops automatically
5. HC always uses the same modset as the server via `current.txt`

### Important Notes

- HC connects as **headlessclient** — this is NOT counted as a player
- Arma 3 briefly logs a disconnect/reconnect during role selection — this is normal and does not trigger HC stop
- Antistasi Ultimate supports exactly ONE headless client
- Always load a mission before the monitor will start the HC

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
3. HC Monitor detects players — starts HC automatically
4. Session runs with HC handling AI
5. Last player disconnects
6. HC Monitor stops HC automatically

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

## HC Monitor — Technical Notes

The monitor uses these key techniques to accurately track player count:

**Regex pattern `[^dis]connected`** — matches `connected` but NOT `disconnected`. This prevents disconnect events from being double-counted as connections.

**Pipe character filter `grep -v "|"`** — all Arma 3 server events use simple format. All Antistasi internal logs contain pipe characters. This cleanly separates real server events from mission script logs.

**Headless client filter `grep -vi "headless"`** — HC connects as `headlessclient` which contains `headless`. This prevents the HC from counting as a real player.

**Baseline on startup** — on startup the monitor reads the log from the last `Game Port: 2302` line forward and calculates current player count. This means the monitor correctly handles players who were already connected before it started.

**Modset awareness** — the `start_hc` function reads `current.txt` at the moment it starts the HC. This means if you switch modsets the HC always loads the correct one.

---

## Verifying HC Monitor Works

Use this diagnostic to test monitor logic without starting it:

```bash
sudo su - steam
bash -c '
LAST_START=$(grep -n "Game Port: 2302" /home/steam/arma3/server/server_log.txt | tail -1 | cut -d: -f1)
echo "Server started at log line: $LAST_START"
echo ""
echo "=== CONNECTIONS ==="
tail -n +"$LAST_START" /home/steam/arma3/server/server_log.txt | grep -E "Player.*[^dis]connected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|"
echo ""
echo "=== DISCONNECTIONS ==="
tail -n +"$LAST_START" /home/steam/arma3/server/server_log.txt | grep "Player.*disconnected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|"
echo ""
echo "=== COUNTS ==="
CONNECTED=$(tail -n +"$LAST_START" /home/steam/arma3/server/server_log.txt | grep -E "Player.*[^dis]connected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|" | wc -l)
DISCONNECTED=$(tail -n +"$LAST_START" /home/steam/arma3/server/server_log.txt | grep "Player.*disconnected" | grep -vi "headless" | grep -v "Antistasi" | grep -v "|" | wc -l)
echo "Connected: $CONNECTED"
echo "Disconnected: $DISCONNECTED"
echo "Net player count: $((CONNECTED - DISCONNECTED))"
'
```

Expected results with no players: Connected 0, Disconnected 0, Net 0.

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

### Kill All Arma Processes

```bash
sudo su - steam
pkill -f arma3server_x64
```
