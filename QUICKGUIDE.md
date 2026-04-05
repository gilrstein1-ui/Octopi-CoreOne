# Quick Guide: OctoPrint + Obico for Prusa Core One

**Condensed version with all steps and commands.** Each section tells you what you're doing and why, then gives you clearly marked blocks to paste or actions to take. For deep explanations, troubleshooting, and an AI-assisted debug prompt, see the [Full Setup Guide](octoprint-core-one-guide.md).

> This setup has been running for months, catching failed prints before the filament hits the fan — and others have followed this guide and got their setups running successfully. I hope this is useful to you, enjoy the OctoPi experience, it's great! If you feel like it, consider [buying me a coffee](https://ko-fi.com/3dzoidberg) :), thanks in advance!
>
> **Questions or discussion?** [Reddit thread](https://www.reddit.com/r/prusa3d/comments/1s0puol/octoprint_with_ai_failure_detection_for_core_one/)

> **This doesn't replace Prusa Connect.** Both work side by side. Obico can still watch and notify you when printing from Prusa Connect — it just can't auto-pause (only OctoPrint-initiated prints can be paused by Obico).

---

### Before You Begin — Placeholders

This guide uses placeholders you need to replace with your actual values:

| Placeholder | What It Is | Required? |
|---|---|---|
| `YOUR_USERNAME` | SSH/Pi username you set during flashing | Yes — used everywhere |
| `YOUR_PI_IP` | Pi's static IP address | Yes |
| `YOUR_CAMERA_IP` | Buddy camera's static IP | Yes (skip if USB or no camera) |
| `YOUR_API_KEY` | OctoPrint global API key | Yes (from step 5) |
| `YOUR_WINDOWS_IP` | Windows PC IP | Optional — backup push only (step 11) |
| `YOUR_WIN_USER` | Windows username | Optional — backup push only (step 11) |

> **💡 Time-saving tip:** Copy this entire guide into a text editor (Notepad, VS Code, etc.) and use **Find & Replace** to swap all placeholders at once. Then you can paste every SSH block directly without editing each time.

> **⚠️ The #1 cause of setup failures is a missed or mistyped placeholder.** If even one `YOUR_USERNAME` is left unchanged inside a service file, that service silently fails on boot. Treat these like the screw types on the Core One build — get one wrong and things don't work. Use Find & Replace, double-check before pasting.

> **💡 PuTTY paste tip:** Paste each SSH block in one quick action (right-click in PuTTY). Don't paste line by line — PuTTY can truncate multi-line blocks, leaving bash stuck at a `>` prompt. If that happens, Ctrl+C and re-paste.

---

## Hardware

> Links are from CanaKit (official Pi supplier) and other trusted stores. Using them supports this guide at no extra cost to you.

| Component | Link |
|---|---|
| **Raspberry Pi 4B** | [Board only](https://amzn.to/4lPTDxY) · [CanaKit Starter Kit](https://amzn.to/4rPVxjw) (includes case, PSU, heatsinks) |
| **Raspberry Pi 5** | [CanaKit Kit](https://amzn.to/4bpTH3M) — overkill for this but works |
| **Power Supply** | [CanaKit 3.5A USB-C for Pi 4](https://amzn.to/4ccWeNT) — needed if buying board only |
| **SD Card** | [SanDisk 64GB High Endurance](https://amzn.to/4rIxiU9) — standard cards wear out fast |
| **USB Cable** | USB-A to USB-C — **tape the 5V pin** ([photo](usb-5v-pin-taped.jpg)) |
| **Camera** | Prusa Buddy Wi-Fi camera (or any RTSP / USB camera) |

---

## Before You Start

*Done on your PC and printer before SSH-ing into the Pi.*

**💻 On your PC:**
1. **Flash OctoPi 64-bit** via Raspberry Pi Imager → click APP OPTIONS → add custom repo `http://unofficialpi.org/rpi-imager/rpi-imager-octopi.json` → select 64-bit OctoPi. Set hostname, username, password, Wi-Fi (can skip if using LAN cable), enable SSH.

**🔌 On your router after booting Pi with the flashed card:**
2. **Assign static IPs** for both the Pi and the Buddy camera. Do this in the router, not on the devices.

**📱 In the Prusa App** (if using Buddy camera):
3. Tap your printer → Camera tab → tap camera → settings cogwheel → toggle **ON** "RTSP stream on local network." Note the camera IP shown — this is `YOUR_CAMERA_IP`. Without this, go2rtc can't connect to the camera.

**🔧 On the USB cable:**
4. **Tape the 5V pin** on the USB-A connector before plugging into the printer (image included in repo). Plug into the Pi's **USB 2.0 port** (black, not blue) — less noise and avoids Wi-Fi interference. USB-C end into the Core One.

**🖨️ On the Core One LCD:**
5. **Older firmware only:** If you see Settings → RPi Port on the LCD, set it to **Off** and power cycle. If the option isn't there, skip this.

---

## 1. First Boot — Verify

*Confirms OS, architecture, and services are correct before you start configuring.*

**⌨️ Paste into SSH:**
```bash
echo "=== SYSTEM ==="
hostname && whoami && uname -m
cat /etc/os-release | grep PRETTY_NAME
free -h | head -2 && df -h / | tail -1
vcgencmd measure_temp && vcgencmd get_throttled
echo "=== OCTOPRINT ==="
systemctl is-active octoprint
echo "=== USB ==="
lsusb
echo "=== CAMERA SERVICES ==="
for svc in webcamd camera-streamer ffmpeg_hls streamer_select; do
  printf "%-20s %s\n" "$svc:" "$(systemctl is-active $svc 2>/dev/null || echo 'not found')"
done
echo "=== PI MODEL ==="
cat /proc/device-tree/model && echo
```

**✅ Verify before moving on:**
- Architecture: `aarch64` (not `armv7l`)
- OctoPrint: `active`
- Throttle: `0x0` (anything else → fix your power supply first)
- Printer visible in `lsusb` output

---

## 2. SD Card Protection

*Moves swap to compressed RAM, reduces filesystem writes, sets up log rotation. Your SD card lasts years instead of months.*

**⌨️ Paste into SSH:**
> ⚠️ **1 placeholder to replace:** `YOUR_USERNAME`

```bash
# ── Disable SD swap (replaced by zram below) ──
sudo dphys-swapfile swapoff
sudo systemctl disable dphys-swapfile
sudo systemctl mask dphys-swapfile

# ── Create zram swap (512MB compressed in RAM, zero SD wear) ──
sudo tee /etc/systemd/system/zram-swap.service > /dev/null << 'ENDZRAM'
[Unit]
Description=Configure zram swap device
After=local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c '\
  modprobe zram && \
  echo lz4 > /sys/block/zram0/comp_algorithm && \
  echo 512M > /sys/block/zram0/disksize && \
  mkswap /dev/zram0 && \
  swapon -p 100 /dev/zram0'
ExecStop=/bin/bash -c '\
  swapoff /dev/zram0 || true && \
  echo 1 > /sys/block/zram0/reset'

[Install]
WantedBy=multi-user.target
ENDZRAM
sudo systemctl daemon-reload
sudo systemctl enable --now zram-swap.service

# ── Fstab tweaks (reduce SD write frequency) ──
if grep -q 'commit=900' /etc/fstab; then
  echo "commit=900 already present"
elif grep -q 'noatime' /etc/fstab; then
  sudo sed -i 's|defaults,noatime|defaults,noatime,commit=900|' /etc/fstab
else
  sudo sed -i 's|\( / .*defaults\)|\1,noatime,commit=900|' /etc/fstab
fi
if ! grep -q '/var/log.*tmpfs' /etc/fstab; then
  echo "tmpfs /var/log tmpfs defaults,noatime,nosuid,nodev,noexec,mode=0755,size=50M 0 0" | sudo tee -a /etc/fstab
fi
sudo tune2fs -c 1 /dev/mmcblk0p2

# ── Persistent journald (crash logs survive reboots, capped at 50MB) ──
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/size-limit.conf > /dev/null << 'ENDJOURNAL'
[Journal]
Storage=persistent
SystemMaxUse=50M
SystemKeepFree=100M
MaxFileSec=1week
ENDJOURNAL
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald

# ── OctoPrint log rotation (prevents logs from filling the card) ──
sudo tee /etc/logrotate.d/octoprint > /dev/null << 'ENDLOGROTATE'
/home/YOUR_USERNAME/.octoprint/logs/octoprint.log {
    daily
    rotate 3
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    size 10M
}
ENDLOGROTATE
```

---

## 3. Disable Default Camera Services *(Buddy camera only)*

*We replace OctoPi's default camera services with go2rtc for the Buddy camera.*

> **🔌 USB camera users: skip steps 3 and 4 entirely.** `webcamd` IS your camera — disabling it kills your feed. Jump to step 5.

**⌨️ Paste into SSH:**
```bash
for svc in webcamd camera-streamer ffmpeg_hls streamer_select; do
  sudo systemctl stop "$svc" 2>/dev/null
  sudo systemctl disable "$svc" 2>/dev/null
  sudo systemctl mask "$svc" 2>/dev/null
done

OCTOPI_CONF="/boot/firmware/octopi.txt"
[ ! -f "$OCTOPI_CONF" ] && OCTOPI_CONF="/boot/octopi.txt"
if [ -f "$OCTOPI_CONF" ]; then
  sudo sed -i 's/^camera_usb_options/#camera_usb_options/' "$OCTOPI_CONF"
  sudo sed -i 's/^camera_http_options/#camera_http_options/' "$OCTOPI_CONF"
  sudo sed -i 's/^camera_auto_start/#camera_auto_start/' "$OCTOPI_CONF"
fi
```

---

## 4. Install go2rtc + Camera Pipeline *(Buddy camera only)*

*Bridges the Buddy camera's RTSP stream to MJPEG for OctoPrint + JPEG snapshots for Obico AI. USB camera users already have this handled by webcamd — skip to step 5.*

**⌨️ Paste into SSH:**
> ⚠️ **11 replacements in this block:** `YOUR_USERNAME`, `YOUR_CAMERA_IP`

```bash
# ── Download go2rtc (single binary, no dependencies) ──
cd /home/YOUR_USERNAME
curl -L -o go2rtc https://github.com/AlexxIT/go2rtc/releases/download/v1.9.9/go2rtc_linux_arm64
chmod +x go2rtc
./go2rtc --version

# ── Config (RTSP path /live is for Buddy camera — others may differ) ──
# 5fps = identical snapshot quality to 25fps, saves ~25% CPU
cat > /home/YOUR_USERNAME/go2rtc.yaml << 'ENDCONF'
streams:
  buddy:
    - rtsp://YOUR_CAMERA_IP/live
  buddy_mjpeg:
    - ffmpeg:buddy#video=mjpeg#raw=-r 5

api:
  listen: "127.0.0.1:1984"

rtsp:
  listen: "127.0.0.1:8554"

ffmpeg:
  bin: ffmpeg

log:
  level: warn
ENDCONF

# ── go2rtc service ──
sudo tee /etc/systemd/system/go2rtc.service > /dev/null << 'ENDSVC'
[Unit]
Description=go2rtc streaming server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=YOUR_USERNAME
Group=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME
ExecStart=/home/YOUR_USERNAME/go2rtc -config /home/YOUR_USERNAME/go2rtc.yaml
Restart=always
RestartSec=5
Nice=5
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/YOUR_USERNAME
PrivateTmp=true

[Install]
WantedBy=multi-user.target
ENDSVC
sudo systemctl daemon-reload
sudo systemctl enable --now go2rtc.service

# ── Keepalive service (holds MJPEG stream open for fast snapshots) ──
sudo tee /etc/systemd/system/go2rtc-keepalive.service > /dev/null << 'ENDSVC2'
[Unit]
Description=go2rtc keepalive (holds MJPEG stream open)
After=go2rtc.service
Requires=go2rtc.service

[Service]
Type=simple
User=YOUR_USERNAME
Group=YOUR_USERNAME
ExecStart=/bin/bash -c 'while true; do curl -s --max-time 86400 http://127.0.0.1:1984/api/stream.mjpeg?src=buddy_mjpeg > /dev/null 2>&1; sleep 2; done'
Restart=always
RestartSec=5
Nice=15

[Install]
WantedBy=multi-user.target
ENDSVC2
sudo systemctl daemon-reload
sleep 3
sudo systemctl enable --now go2rtc-keepalive.service

# ── Point haproxy webcam proxy at go2rtc ──
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo sed -i 's|server webcam1 .*|server webcam1 127.0.0.1:1984|' /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy

# ── Verify ──
sleep 5
for i in 1 2 3; do
  curl -s -o /dev/null -w "grab $i: %{http_code}, %{size_download}B, %{time_total}s\n" \
    "http://127.0.0.1:1984/api/frame.jpeg?src=buddy_mjpeg"
done
```

**✅ Expected:** All three grabs should be HTTP 200, ~120-130KB, <0.3 seconds. If >1 second, check `systemctl status go2rtc-keepalive`.

---

## 5. OctoPrint First Setup

*Configure OctoPrint in your browser — webcam URLs, printer profile, and grab your API key.*

**🌐 In your browser** at `http://YOUR_PI_IP`:

1. Complete setup wizard → create admin account
2. Set webcam URLs *(Buddy camera only)*:
   - **Stream URL:** `/webcam/api/stream.mjpeg?src=buddy_mjpeg`
   - **Snapshot URL:** `http://127.0.0.1:1984/api/frame.jpeg?src=buddy_mjpeg`

> **🔌 USB camera users:** Don't touch webcam URLs — the defaults are already correct. Just verify you see a live feed in OctoPrint's Control tab.

3. Create printer profile:
   - Name: **Prusa Core One** · Model: **COREONE**
   - Build volume: **250 × 220 × 270mm**
   - Heated bed: **Yes** · Heated chamber: **Yes**
   - Nozzle: **0.4mm**
   - Axes jog speeds (mm/min): X: **21000** · Y: **21000** · Z: **1200** · E: **6000**
4. **Delete the `_default` profile** (trash icon next to it)
5. **Copy your API key:** Settings → API → Global API Key → Copy

> This is `YOUR_API_KEY` — you'll need it for validation and backup scripts. **Don't share it publicly** — it grants full control of your OctoPrint instance.

---

## 6. Connect Printer

*USB serial connection at 230400 baud. Higher baudrate prevents buffer underruns on fast CoreXY moves.*

**🌐 In OctoPrint** → Connection panel:
- Serial Port: `/dev/ttyACM0` · Baudrate: `230400`
- Check **Auto-connect on server startup** → Click **Connect**

**⌨️ Then paste into SSH** (low-latency serial rule):
```bash
sudo tee /etc/udev/rules.d/99-usb-serial.rules > /dev/null << 'ENDUDEV'
ACTION=="add", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{idProduct}=="001f", RUN+="/bin/setserial /dev/%k low_latency"
ENDUDEV
sudo udevadm control --reload-rules
```

**✅ Expected:** OctoPrint shows "Operational" with baudrate 230400. Udev rule takes effect on next reboot.

---

## 7. Install Plugins *(all optional)*

*Search and install each in OctoPrint UI. On a 2GB Pi, install one at a time with 30s pauses.*

**🌐 In OctoPrint:** Settings → Plugin Manager → Get More → search each name:

| Priority | Plugins |
|---|---|
| **Mission-critical** | Obico for OctoPrint · Cancel Objects · HeaterTimeout |
| **Recommended** | PrintTimeGenius · Slicer Thumbnails · Display Layer Progress · Navbar Temp |
| **Nice to have** | Dashboard · UI Customizer · Backup Schedule ⚠️ |

> ⚠️ Backup Schedule v0.2.6 has a known bug (`InvalidToken` errors). See full guide if affected.

---

## 8. G-code Scripts

*Without these, pausing a print (or Obico auto-pausing) leaves the hot nozzle melting into your print.*

**🌐 In OctoPrint:** Settings → G-code Scripts:

| Event | Script | What it does |
|---|---|---|
| After print job is **paused** | `M601` | Firmware parks nozzle, manages temp |
| Before print job is **resumed** | `M602` | Firmware returns to position, reheats |
| After print job is **cancelled** | `M603` | Firmware cleans up, retracts, homes |

**🌐 Also enable:** Settings → Serial Connection → Behaviour → Pausing → **Log position on pause**

**🌐 Optional — bed drop after print** (G-code Scripts → After print job is done):
```
{% if last_position.z is not none and last_position.z < 135 %}
G1 Z135 F720
{% endif %}
```
> Drops bed to mid-height on prints <135mm for easy part removal. Taller prints left as-is.

---

## 9. Link Obico

*AI failure detection — watches your camera and auto-pauses on spaghetti/blob-of-death. Obico reads webcam URLs from OctoPrint automatically — you don't configure camera URLs in Obico.*

**🌐 In OctoPrint:**
1. Settings → Obico for OctoPrint → **Run Setup Wizard** → Obico Cloud
2. On phone: Obico app → Link Printer → enter 6-digit code in OctoPrint wizard
3. In Obico plugin settings: Resolution **Medium** · FPS **5** · Action on failure **Pause + Notify**

> Resolution and FPS control the **live viewing stream** in the Obico app only — they do NOT affect AI detection quality. The AI grabs full-resolution snapshots separately.

> If linking fails (loops back to "not linked"), see full guide for the config-clearing fix.

**⌨️ Verify snapshot speed** *(Buddy camera)*:
```bash
curl -s -o /dev/null -w "HTTP %{http_code}, %{size_download}B, %{time_total}s\n" \
  "http://127.0.0.1:1984/api/frame.jpeg?src=buddy_mjpeg"
```

**⌨️ Verify snapshot speed** *(USB camera)*:
```bash
curl -s -o /dev/null -w "HTTP %{http_code}, %{size_download}B, %{time_total}s\n" \
  "http://127.0.0.1:8080/?action=snapshot"
```

**✅ Expected:** HTTP 200, <0.3 seconds.

---

## 10. PrusaSlicer

*Create a separate printer profile for OctoPrint. Keep your Prusa Connect profile unchanged.*

**💻 In PrusaSlicer** — duplicate your Core One profile, then change:

| Where | Setting | Value |
|---|---|---|
| Printer Settings → General | binary_gcode | **Off** |
| Printer Settings → G-code thumbnails | *(replace entire line)* | `16x16/QOI, 313x173/QOI, 480x240/QOI, 380x285/PNG, 32x32/PNG, 400x300/PNG` |
| Print Settings → Output Options | Label objects | **OctoPrint comment** |
| Print Settings → Advanced | Arc fitting | **Enabled** |

**💻 Add Physical Printer** (cog icon next to Printer dropdown):
- Host Type: **OctoPrint** · Hostname: `YOUR_PI_IP` · API Key: *(generate in Settings → Application Keys)*

> ⚠️ Label Objects and Arc Fitting are per **print profile** — enable in each profile you use.

> ⚠️ Always use **"Load model only"** when opening 3MF files to prevent profile overrides.

---

## 11. Backups *(Optional)*

*System backup script captures everything OctoPrint's built-in backup misses. Runs nightly via cron. Optionally pushes to a Windows PC.*

**⌨️ Paste into SSH:**
> ⚠️ **18 replacements in this block:** `YOUR_USERNAME`, `YOUR_WINDOWS_IP`, `YOUR_WIN_USER`

```bash
# ── Install SMB client (only needed if pushing to Windows) ──
sudo apt-get install -y cifs-utils
sudo mkdir -p /mnt/winbackup

# ── Create backup script ──
cat > /home/YOUR_USERNAME/backup.sh << 'ENDSCRIPT'
#!/bin/bash
BACKUP_DIR="/home/YOUR_USERNAME/backups"
WIN_SHARE="//YOUR_WINDOWS_IP/YourShareName"
WIN_MOUNT="/mnt/winbackup"
DATE=$(date +%Y-%m-%d)
DAY_OF_WEEK=$(date +%u)
DAY_OF_MONTH=$(date +%d)
BACKUP_NAME="backup-${DATE}.tar.gz"
LOG_NAME="logs-${DATE}.tar.gz"
mkdir -p "$BACKUP_DIR"
crontab -l > /home/YOUR_USERNAME/crontab.bak 2>/dev/null
tar czf "${BACKUP_DIR}/${BACKUP_NAME}" \
  /home/YOUR_USERNAME/go2rtc /home/YOUR_USERNAME/go2rtc.yaml \
  /home/YOUR_USERNAME/backup.sh /home/YOUR_USERNAME/crontab.bak \
  /home/YOUR_USERNAME/.octoprint/config.yaml \
  /home/YOUR_USERNAME/.octoprint/printerProfiles/ \
  /home/YOUR_USERNAME/.octoprint/data/backup/ \
  /etc/systemd/system/go2rtc.service /etc/systemd/system/go2rtc-keepalive.service \
  /etc/systemd/system/zram-swap.service /etc/haproxy/haproxy.cfg \
  /etc/fstab /etc/logrotate.d/octoprint /etc/systemd/journald.conf.d/ \
  /boot/firmware/config.txt 2>/dev/null
[ "$DAY_OF_WEEK" -eq 7 ] && cp "${BACKUP_DIR}/${BACKUP_NAME}" "${BACKUP_DIR}/weekly-${DATE}.tar.gz"
[ "$DAY_OF_MONTH" -eq "01" ] && cp "${BACKUP_DIR}/${BACKUP_NAME}" "${BACKUP_DIR}/monthly-${DATE}.tar.gz"
find "${BACKUP_DIR}" -name "backup-*.tar.gz" -mtime +7 -delete
find "${BACKUP_DIR}" -name "weekly-*.tar.gz" -mtime +28 -delete
LOG_TMP=$(mktemp -d)
cp /home/YOUR_USERNAME/.octoprint/logs/*.log "${LOG_TMP}/" 2>/dev/null
journalctl -u octoprint --since "24 hours ago" --no-pager > "${LOG_TMP}/journal-octoprint.log" 2>/dev/null
journalctl -u go2rtc --since "24 hours ago" --no-pager > "${LOG_TMP}/journal-go2rtc.log" 2>/dev/null
journalctl -u go2rtc-keepalive --since "24 hours ago" --no-pager > "${LOG_TMP}/journal-keepalive.log" 2>/dev/null
journalctl -u haproxy --since "24 hours ago" --no-pager > "${LOG_TMP}/journal-haproxy.log" 2>/dev/null
journalctl --since "24 hours ago" --priority=err --no-pager > "${LOG_TMP}/journal-errors.log" 2>/dev/null
echo "Date: ${DATE}" > "${LOG_TMP}/system-stats.txt"
echo "Uptime: $(uptime)" >> "${LOG_TMP}/system-stats.txt"
free -h >> "${LOG_TMP}/system-stats.txt"
df -h >> "${LOG_TMP}/system-stats.txt"
vcgencmd measure_temp >> "${LOG_TMP}/system-stats.txt"
vcgencmd get_throttled >> "${LOG_TMP}/system-stats.txt"
tar czf "${BACKUP_DIR}/${LOG_NAME}" -C "${LOG_TMP}" . 2>/dev/null
rm -rf "${LOG_TMP}"
[ "$DAY_OF_MONTH" -eq "01" ] && cp "${BACKUP_DIR}/${LOG_NAME}" "${BACKUP_DIR}/monthly-logs-${DATE}.tar.gz"
find "${BACKUP_DIR}" -name "logs-*.tar.gz" -mtime +7 -delete
if command -v mount.cifs &>/dev/null; then
  sudo mkdir -p "$WIN_MOUNT"
  if sudo mount -t cifs "$WIN_SHARE" "$WIN_MOUNT" \
    -o username=YOUR_WIN_USER,password=,vers=3.0,uid=YOUR_USERNAME,gid=YOUR_USERNAME 2>/dev/null; then
    mkdir -p "${WIN_MOUNT}/Backups/Daily" "${WIN_MOUNT}/Backups/Weekly" "${WIN_MOUNT}/Backups/Monthly"
    mkdir -p "${WIN_MOUNT}/Logs/Daily" "${WIN_MOUNT}/Logs/Monthly"
    mkdir -p "${WIN_MOUNT}/Timelapses"
    cp "${BACKUP_DIR}/${BACKUP_NAME}" "${WIN_MOUNT}/Backups/Daily/" 2>/dev/null
    [ -f "${BACKUP_DIR}/weekly-${DATE}.tar.gz" ] && cp "${BACKUP_DIR}/weekly-${DATE}.tar.gz" "${WIN_MOUNT}/Backups/Weekly/"
    [ -f "${BACKUP_DIR}/monthly-${DATE}.tar.gz" ] && cp "${BACKUP_DIR}/monthly-${DATE}.tar.gz" "${WIN_MOUNT}/Backups/Monthly/"
    cp "${BACKUP_DIR}/${LOG_NAME}" "${WIN_MOUNT}/Logs/Daily/" 2>/dev/null
    [ -f "${BACKUP_DIR}/monthly-logs-${DATE}.tar.gz" ] && cp "${BACKUP_DIR}/monthly-logs-${DATE}.tar.gz" "${WIN_MOUNT}/Logs/Monthly/"
    TIMELAPSE_DIR="/home/YOUR_USERNAME/.octoprint/timelapse"
    if [ -d "$TIMELAPSE_DIR" ] && ls "$TIMELAPSE_DIR"/*.mp4 1>/dev/null 2>&1; then
      for f in "$TIMELAPSE_DIR"/*.mp4; do
        fname=$(basename "$f")
        [ ! -f "${WIN_MOUNT}/Timelapses/${fname}" ] && cp "$f" "${WIN_MOUNT}/Timelapses/"
      done
    fi
    find "${WIN_MOUNT}/Backups/Daily" -name "*.tar.gz" -mtime +30 -delete 2>/dev/null
    find "${WIN_MOUNT}/Logs/Daily" -name "*.tar.gz" -mtime +30 -delete 2>/dev/null
    sudo umount "$WIN_MOUNT" 2>/dev/null
    echo "$(date): Push OK" >> "${BACKUP_DIR}/backup.log"
  else
    echo "$(date): Local OK (Windows offline)" >> "${BACKUP_DIR}/backup.log"
  fi
fi
ENDSCRIPT
chmod +x /home/YOUR_USERNAME/backup.sh

# ── Schedule daily at 3:30 AM (sudo needed for SMB mount) ──
sudo bash -c '(crontab -l 2>/dev/null; echo "30 3 * * * /home/YOUR_USERNAME/backup.sh") | crontab -'
```

**🌐 If you installed Backup Schedule plugin:** Settings → Backup Schedule → set time (e.g., 4:00 AM).

---

## 12. Fan Control *(Optional)*

*GPIO on/off fan. 70°C threshold = safety net only. Normal printing runs 52-65°C, so the fan rarely triggers.*

Wire: Red → pin 4 (5V) · Black → pin 6 (GND) · Blue → pin 8 (GPIO14)

**⌨️ Paste into SSH:**
```bash
echo "dtoverlay=gpio-fan,gpiopin=14,temp=70000" | sudo tee -a /boot/firmware/config.txt
```

> Takes effect on reboot (next step).

---

## 13. Reboot & Validate

*Apply fstab changes and fan overlay. Verify everything comes up clean.*

**⌨️ Paste into SSH:**
```bash
sudo reboot
```

**⌨️ After reconnecting (~60 seconds), paste the block for your camera type:**

**Buddy camera:**
> ⚠️ **1 placeholder to replace:** `YOUR_API_KEY`

```bash
echo "=== SERVICES ==="
for svc in octoprint go2rtc go2rtc-keepalive zram-swap haproxy; do
  printf "%-22s %s\n" "$svc:" "$(systemctl is-active $svc)"
done
echo -e "\n=== PRINTER ==="
API_KEY="YOUR_API_KEY"
curl -s "http://127.0.0.1:5000/api/connection" \
  -H "X-Api-Key: $API_KEY" | python3 -c "
import sys,json; d=json.load(sys.stdin)['current']
print(f'State: {d[\"state\"]}, Baud: {d[\"baudrate\"]}')"
echo -e "\n=== SNAPSHOTS ==="
for i in 1 2 3; do
  curl -s -o /dev/null -w "grab $i: %{http_code}, %{time_total}s\n" \
    "http://127.0.0.1:1984/api/frame.jpeg?src=buddy_mjpeg"
done
echo -e "\n=== SYSTEM ==="
vcgencmd measure_temp && vcgencmd get_throttled
free -h | grep -E "Mem|Swap"
```

**USB camera:**
> ⚠️ **1 placeholder to replace:** `YOUR_API_KEY`

```bash
echo "=== SERVICES ==="
for svc in octoprint webcamd zram-swap haproxy; do
  printf "%-22s %s\n" "$svc:" "$(systemctl is-active $svc)"
done
echo -e "\n=== PRINTER ==="
API_KEY="YOUR_API_KEY"
curl -s "http://127.0.0.1:5000/api/connection" \
  -H "X-Api-Key: $API_KEY" | python3 -c "
import sys,json; d=json.load(sys.stdin)['current']
print(f'State: {d[\"state\"]}, Baud: {d[\"baudrate\"]}')"
echo -e "\n=== SNAPSHOTS ==="
for i in 1 2 3; do
  curl -s -o /dev/null -w "grab $i: %{http_code}, %{time_total}s\n" \
    "http://127.0.0.1:8080/?action=snapshot"
done
echo -e "\n=== SYSTEM ==="
vcgencmd measure_temp && vcgencmd get_throttled
free -h | grep -E "Mem|Swap"
```

**✅ Expected:**
- All services: `active`
- Printer: `Operational`, `230400`
- Snapshots: <0.3s, HTTP 200
- Throttle: `0x0`
- Swap: shows zram (not SD)

---

## Quick Reference

*Day-to-day commands. Paste into SSH when you need them.*

| What | When to use | Command |
|---|---|---|
| **Service status** | After reboot, or if something seems off | Buddy: `for svc in octoprint go2rtc go2rtc-keepalive zram-swap haproxy; do printf "%-22s %s\n" "$svc:" "$(systemctl is-active $svc)"; done` · USB: replace `go2rtc go2rtc-keepalive` with `webcamd` |
| **Snapshot speed** | If Obico seems slow or camera feed is stale | Buddy: `curl -s -o /dev/null -w "%{time_total}s\n" "http://127.0.0.1:1984/api/frame.jpeg?src=buddy_mjpeg"` · USB: `curl -s -o /dev/null -w "%{time_total}s\n" "http://127.0.0.1:8080/?action=snapshot"` |
| **Pi health** | Routine check (throttle should be 0x0) | `vcgencmd measure_temp && vcgencmd get_throttled && free -h` |

**⌨️ Restart camera pipeline** (frozen feed or slow snapshots):
```bash
# Buddy camera:
sudo systemctl restart go2rtc && sleep 2 && sudo systemctl restart go2rtc-keepalive
# USB camera:
sudo systemctl restart webcamd
```

**⌨️ Restart OctoPrint** (unresponsive web UI — won't interrupt active print):
```bash
sudo systemctl restart octoprint
```

**⌨️ Run backup manually** (before making system changes):
> ⚠️ **2 replacements in this block:** `YOUR_USERNAME`

```bash
sudo /home/YOUR_USERNAME/backup.sh && tail -1 /home/YOUR_USERNAME/backups/backup.log
```

**⌨️ Check crash logs** (after unexpected reboot):
```bash
journalctl -b -1 --priority=err --no-pager | head -50
```

---

*This Quick Guide covers the same steps as the [Full Setup Guide](octoprint-core-one-guide.md) — just without the deep explanations. For troubleshooting, architecture decisions, and an AI-assisted debug prompt, see the full guide.*
