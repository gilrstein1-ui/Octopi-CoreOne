# OctoPrint + Obico Setup Guide for Prusa Core One

A complete, tested, production-ready setup for running OctoPrint with AI failure detection on a Prusa Core One.

![Raspberry Pi 4B · OctoPi 64-bit · go2rtc · Obico AI](https://img.shields.io/badge/Pi_4B-OctoPi_64--bit-red) ![go2rtc 1.9.9](https://img.shields.io/badge/go2rtc-1.9.9-blue) ![Obico AI](https://img.shields.io/badge/Obico-AI_Detection-green)

I worked hard to end up with what feels like a very reliable solution which has been working for months and has managed to catch and pause failed prints before the filament hits the fan (a little poop joke for you right there). I saw repeating questions on Reddit which I felt this solution might be great for, and since this info was not all readily available online, I decided to close the gap and make this guide.

This is my first time including affiliate links to Amazon — curious to see what happens. Feel free to ask any questions and I hope it serves you well. If you want to send a little something my way, [ko-fi.com/3dzoidberg](https://ko-fi.com/3dzoidberg) — thank you in advance :)

---

## What This Gets You

- **OctoPrint** controlling your Core One over USB at 230400 baud
- **AI failure detection** — Obico watches your prints and auto-pauses on spaghetti / blob-of-death
- **Remote monitoring** from your phone via the Obico app
- **Live 1080p camera feed** with ~0.2 second snapshots through go2rtc
- **PrusaSlicer direct upload** with thumbnails and cancel-object support
- **SD card protection** — zram swap, tmpfs logs, reduced writes
- **Automated tiered backups** with optional push to a Windows PC
- **Everything auto-starts on boot** — zero manual intervention after power-on

## Guides

| Guide | For Who | Time |
|---|---|---|
| [**Full Setup Guide**](octoprint-core-one-guide.md) | First-time setup — every step explained with context and reasoning | 2–3 hours |
| [**Quick Guide**](QUICKGUIDE.md) | Experienced users or re-doing a setup after a reflash — condensed steps with all the commands | 30–45 min |

Start with the **Full Setup Guide** if this is your first time. Use the **Quick Guide** for subsequent setups or as a reference checklist.

## Architecture

```
Prusa Buddy Camera (Wi-Fi, RTSP)
       │
       ▼
   go2rtc (localhost:1984)
       │
       ├──► MJPEG stream → haproxy → OctoPrint live view
       ├──► JPEG snapshots → Obico AI detection
       └──► RTSP restream (localhost:8554)
                │
                ▼
         Obico plugin → H.264 re-encode → cloud / mobile app

   OctoPrint (localhost:5000) ──USB serial 230400──► Prusa Core One (xBuddy board)
       │
       └──► haproxy (port 80) → browser
```

## Hardware

> Using the links below supports this guide at no extra cost to you. You can also support directly at [ko-fi.com/3dzoidberg](https://ko-fi.com/3dzoidberg).

| Component | Recommendation |
|---|---|
| **Raspberry Pi** | [Pi 4B — Official Store](https://amzn.to/4lPTDxY) · [Pi 4B CanaKit](https://amzn.to/4rPVxjw) · [Pi 5 CanaKit](https://amzn.to/4bpTH3M) |
| **SD Card** | [SanDisk 64GB High Endurance](https://amzn.to/4rIxiU9) — standard cards wear out fast |
| **USB Cable** | USB-A to USB-C — **tape the 5V pin** ([see photo](usb-5v-pin-taped.jpg)) |
| **Camera** | Prusa Buddy Wi-Fi camera (or any RTSP camera) |

This setup runs flawlessly on a **Pi 4B** — that's what I use daily. A Pi 5 works but is overkill. I recommend buying a kit with case + fan + power supply bundled — more cost-effective than separate parts.

## How This Guide Was Built

Every SSH command, service file, and config block was developed interactively with Claude Opus. I planned, tested, verified, and looked up everything extensively before committing each change.

The **Troubleshooting** section of the full guide includes a ready-to-paste prompt that equips Claude with complete context about this system architecture — so if you hit an issue, you can get targeted help immediately.

## Repo Contents

```
├── README.md                      ← You are here
├── octoprint-core-one-guide.md    ← Full setup guide (1,300+ lines)
├── QUICKGUIDE.md                  ← Quick Guide — condensed steps with commands
├── rpi-imager-app-options.jpg     ← Screenshot: RPi Imager APP OPTIONS button
└── usb-5v-pin-taped.jpg           ← Photo: USB 5V pin taped
```
