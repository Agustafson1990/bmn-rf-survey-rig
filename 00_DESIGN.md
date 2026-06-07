> **Redaction note.** This file has been programmatically sanitized for public
> publication: maintainer agent names, internal workflow references, internal
> filesystem paths, internal IPs, and project-meta cross-references have been
> stripped or generalized. The technical content (RF design, Wi-Fi guardrails,
> portal mechanics, hardware layout) is intact. See the repo README for the
> public-facing project overview.

# 00_DESIGN — BMN RF Survey Rig (w/ DefCon Mode)

**Document type:** Engineering design (pre-implementation).
**Authored by:** the maintainer.
**For review by:** the maintainer (Owner) and the architect (PM/Architect).
**Drafted:** 2026-05-25.
**Supersedes:** `00_DESIGN_v1.md` (frozen reference — the original single-role captive-portal-only design from 2026-04-27). v1 carried forward verbatim where called out below; everything else is new.
**Status:** Draft 1 of v2 — no code or RouterOS configs produced. Awaiting the maintainer's answers in §14 before queueing any (task queue) for build steps.

This document is the engineering design for the dual-role rig. The rig has **two roles**:

1. **Always-on:** Portable RF discovery + monitoring + recording appliance. Two RTL-SDR-compatible dongles on a Pi 4, scanning legally-monitorable bands, recording audio bursts, detecting CTCSS/DCS tones, surfacing a phone-friendly web UI for browser-based monitoring + live audio (OpenWebRX).
2. **Toggleable:** DefCon-style captive-portal Rickroll prank from v1, with the broadcast and portal logic now living on the MikroTik mAP lite's 2.4 GHz radio (`wlan2`) — disabled by default; flipped on by a tik-side script or a Pi web-UI button calling a scoped RouterOS REST user.

Read **§13b (RF-safety boundaries)** and **§13 (wifi-safety boundaries, cite v1)** before any other section if you are skimming — those guardrails dictate every decision in §1–§12.

---

## §0 — Briefs (verbatim, reconciled)

### §0.1 — the architect PM brief (RF survey appliance, the always-on role)

> Build a Raspberry Pi 4 portable RF discovery and recording appliance using two RTL-SDR compatible dongles.
>
> Hardware: Raspberry Pi 4, Raspberry Pi OS 64-bit, two NooElec RTL-SDR dongles, USB battery bank.
>
> Primary objective: on boot, automatically begin scanning configured non-broadcast radio ranges, detect active analog FM voice channels, record audio bursts, detect CTCSS/DCS squelch tones/codes when possible, and display results in a simple local web UI.
>
> Architecture: SDR #1 = discovery scanner; SDR #2 = locked monitor/recorder for active or promoted channels. Lightweight RTL-SDR tools, not SDRTrunk as primary engine. Python Flask or FastAPI web interface. SQLite for event logging. systemd autostart on boot. Optional local streaming with Icecast / simple HTTP audio stream.
>
> Required web UI: dashboard (active freq, last heard, CTCSS/DCS, level, duration, recording link), scan range editor, discovered frequency table (hit count, last heard, tone/code, promote-to-monitor button), recordings browser (play/download), service status (SDR health, CPU temp, disk space, battery/runtime estimate), buttons for restart scanner/recorder/reload config.
>
> Data: scan ranges in `/opt/bmn-rf/config/scan-ranges.json`; channels in `/opt/bmn-rf/config/channels.json`; recordings in `/opt/bmn-rf/recordings/YYYY-MM-DD/`; logs in `/opt/bmn-rf/logs/`; SQLite at `/opt/bmn-rf/db/rf_events.sqlite`.
>
> Recording: only when carrier/audio active. Filename `YYYY-MM-DD_HH-MM-SS_frequency_tone_or_CSQ_duration.wav`. America/Chicago local time. Each transmission its own clip.
>
> Detection goals: frequency, signal strength, CTCSS tone, DCS code, duration, timestamp, SDR device, audio file path.
>
> Default scan presets: Ham VHF 144–148 MHz, Ham UHF 420–450 MHz, MURS 151/154, GMRS/FRS 462/467, Business VHF/UHF common, NOAA WX, Airband RX if AM practical.
>
> Constraints: ease of operation critical. Auto-start on power-on. Phone-usable web UI. No public internet exposure. Local LAN only. Reliability over fancy UI. Install script + systemd units + README + troubleshooting steps.

### §0.2 — Claude-chat synopsis (architecture wrap from the maintainer's RF-build conversation)

The architecture wrap that arrived alongside §0.1 reaches the same RF goal but specifies the hosting topology: Pi 4 (4 GB) carries SDRs + thumb drive + OpenWebRX + survey scanner + web UI; **MikroTik mAP lite handles all routing**, multi-SSID broadcasting (for the DefCon mode), captive portal mechanics, and a WireGuard tunnel back to BMN's BMN VPN hub. Pi's `wlan0` = mgmt AP for the maintainer's phone + workflow. Pi's `eth0` = LAN to tik. SDRs and thumb drive on Pi USB ports (no hub). One battery brick, two cables (Pi + tik). DefCon mode = wlan2 (2.4 GHz AP) enabled + captive portal active on tik; toggle via shell scripts on tik OR a Pi web-UI button calling a restricted RouterOS REST API user (write access scoped to `/interface/wireless` and `/ip/hotspot` only).

WireGuard lives on the tik (not the Pi) — Pi has no WG awareness; just routes BMN-destined packets to the tik which encrypts and tunnels. Default state: BMN VPN hub firewall DROPS inbound WG handshake, so the tunnel is functionally closed even with keys present on both sides. the maintainer opens the firewall on BMN VPN hub when access is needed; tunnel negotiates in ~25 s via PersistentKeepalive; closes when locked again. Rig cannot phone home unless BMN explicitly allows it.

Defense-in-depth: (1) tik firewall segments captive portal subnet from mgmt, (2) Pi nftables restricts `eth0`-facing exposure to only the rickroll endpoint, (3) WG controlled by BMN VPN hub firewall, (4) Pi→tik REST credential locked to a RouterOS user group with write access only to the captive portal interfaces.

### §0.3 — Original v1 brief (carry-forward)

> Build a portable Wi-Fi prank/demo rig using: MikroTik mAP lite as AP/router/captive portal controller; Raspberry Pi 4B as local web server, captive portal host, media host, logger, database server, admin dashboard, and export/upload worker; battery bank for portable operation. mAP lite broadcasts multiple open SSIDs; client redirected to captive portal hosted on the Pi; captive portal plays locally-hosted Rick Astley video; page references `https://social.bytemenetworks.com` and serves a local mirror/fallback when offline. Connection metrics logged locally; opportunistic NAS upload when Pi sees internet. Admin page: view metrics, export, manual upload, reset, upload history, health.
>
> Hard guardrails: not credential harvesting, not phishing, not malware, not traffic interception. Don't impersonate trusted secure networks. Don't capture passwords. Funny, portable, technically clean.

### §0.4 — Reconciliation: what changed from v1 → v2

| v1 (single-role) | v2 (dual-role) |
|---|---|
| mAP lite always broadcasting open SSIDs | mAP lite's wlan2 (2.4 GHz AP for captive portal) is **disabled by default**; wlan1 (5 GHz STA) used for opportunistic WAN uplink to known SSIDs (home Wi-Fi, hotspot, hotel) |
| Pi captive-portal serving is the entire purpose | Captive-portal serving is the **toggleable** purpose; RF survey is the always-on purpose |
| No SDRs | Two NooElec SDRs on Pi USB3; thumb drive on USB2 for DB + recordings (saves SD wear) |
| WAN to Pi via Pi `wlan0` (operator mode) or USB tether | WAN to mAP lite via tik `wlan1` (5 GHz STA mode, priority SSID list). Pi has no WAN responsibility; just LAN to tik via `eth0`. Pi `wlan0` is mgmt AP only (phone, workflow, SSH). |
| NAS upload triggered by opportunistic Pi-side internet | NAS upload triggered by tik-side WireGuard window (BMN VPN hub opens firewall, tunnel negotiates, Pi rsyncs through tik tunnel, firewall closes) |
| No tone detection | CTCSS / DCS detection required in v1 of the RF role |
| Single SQLite for portal hits | Two databases: portal SQLite (`/var/lib/bmn-rfrig/portal/rickroll.db`) carries forward from v1; new RF SQLite (`/opt/bmn-rf/db/rf_events.sqlite`) per the architect brief, on the thumb drive |
| Hardware: Pi 4B + mAP lite + battery | Hardware adds 2 SDRs + thumb drive; battery sizing revised upward (§11) |
| §13 wifi-safety boundaries only | §13 (wifi) carries forward unchanged; **new §13b RF-safety boundaries** added |

**Paths:** v1 used `/opt/rickroll/`. v2 unifies under a single umbrella `/opt/bmn-rfrig/` for the rig as a whole, while preserving the architect's brief-specified `/opt/bmn-rf/` namespace for the RF role's config + DB + recordings (since it's the namespace the brief explicitly called out). See §2 filesystem layout.

---

## §1 — Recommended technical architecture (dual-role)

### §1.1 — Hardware

| Component | Role | Notes |
|---|---|---|
| **Raspberry Pi 4B (4 GB)** | Brain: SDR survey scanner + recorder, OpenWebRX, captive portal host (when DefCon Mode on), nginx, web UI, SQLite, systemd glue | 64-bit Raspberry Pi OS Lite Bookworm; case the maintainer-provided; ambient-limited cooling accepted |
| **2 × NooElec RTL-SDR** | RF receive | One always-on for OpenWebRX live audio; one for the survey scanner (discovery + locked-monitor as PM brief spec'd). 6" right-angle USB3 jumpers for clean stacking |
| **USB thumb drive** | RF SQLite DB + OpenWebRX logs + RF recordings | Off-loads write churn from the Pi SD card |
| **MikroTik mAP lite** | All routing, captive portal controller (when DefCon Mode on), WAN uplink (5 GHz STA), WireGuard endpoint to BMN VPN hub | Spare unit the maintainer owns. USB-powered. RouterOS 7.x. Subnets: mgmt (internal IP)/24, captive portal (internal IP)/24 |
| **Antennas (two)** | RF capture — Option B per 2026-05-25 the maintainer decision: two dedicated, each optimized | Candidate set: collapsible discone (Sigma ~30 cm packed) for wideband survey scanner; targeted wideband mag-mount mobile whip (e.g., Tram 1410, 25–1300 MHz) for monitor/recorder. Final selection pinned at parts-in-hand time. |
| **Battery: Anker PowerHouse-equivalent** (overnight) + small banks (DefCon travel) | Power both Pi and tik off one brick | Two cables off one battery: USB-A → USB-C → Pi; USB-A → microUSB → mAP lite. Skips USB-C PD negotiation (dumb 5 V more reliable on Pi 4). |

### §1.2 — USB topology on the Pi (no hub)

```
Pi 4B
 ├ USB3 port 1 → SDR #1 (always-on / OpenWebRX live)
 ├ USB3 port 2 → SDR #2 (survey scanner: hop-and-detect; promote → locked monitor)
 ├ USB2 port 1 → thumb drive (RF DB + recordings)
 └ USB2 port 2 → free (operator USB-tether option, future)
```

No hub. 6" right-angle USB3 jumpers keep the SDRs from sticking out awkwardly.

### §1.3 — Network topology

```
                   ┌──────────────────────────────┐
                   │  BMN VPN hub (Vultr, BMN)    │
                   │  WG server, firewall-gated   │
                   └──────────────┬───────────────┘
                                  │ (WG handshake when BMN firewall opens)
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│  MikroTik mAP lite (RouterOS 7.x)                                      │
│                                                                        │
│  wlan1 (5 GHz, STA mode)  ← priority SSID list (Toaster, phone HS,     │
│                              hotel wifi, etc.) → WAN uplink, NAT       │
│                              to LAN                                    │
│  wlan2 (2.4 GHz, AP mode) → captive portal SSIDs — DISABLED BY DEFAULT │
│                              Enabled only by enable-defcon.sh or       │
│                              the Pi web-UI toggle (REST API call).     │
│  ether1 (wired WAN backup) → alternate WAN-in                          │
│  ether2 (LAN)             → to Pi eth0 ((internal IP)/24 mgmt subnet)   │
│  wg0                      → WireGuard client to BMN VPN hub,           │
│                              AllowedIPs scoped to BMN home network     │
│                              only — NOT 0.0.0.0/0.                     │
│                                                                        │
│  Subnets (logical):                                                    │
│    mgmt           (internal IP)/24  (Pi + the maintainer's devices on wlan0/eth) │
│    captive portal (internal IP)/24  (isolated, when DefCon Mode on)     │
└────────────────────────────────────────────────────────────────────────┘
                                  │ ether2
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Raspberry Pi 4B (Raspberry Pi OS Lite, Bookworm 64-bit)               │
│                                                                        │
│  eth0   = (internal IP)/24 — LAN to tik. Only listens for /rickroll/   │
│           (when DefCon Mode on) + Pi REST proxy traffic for captive    │
│           portal subnet's walled garden.                               │
│  wlan0  = built-in BCM43455 — mgmt AP, WPA2. the maintainer's phone,            │
│           workflow agents, SSH, OpenWebRX, scanner UI, DefCon toggle     │
│           button — ALL bind here only.                                 │
│  USB    = 2 × SDR + thumb drive (see §1.2)                             │
│                                                                        │
│  Services:                                                             │
│    - OpenWebRX (port 8073)  → live browser SDR (SDR #1)                │
│    - bmn-rf-scanner         → survey/discovery + locked monitor (#2)   │
│    - bmn-rf-tone-detector   → CTCSS/DCS demodulation                   │
│    - bmn-rf-webui (Flask)   → scan ranges, discovered freqs, recs      │
│    - bmn-portal (Flask)     → captive portal app (v1 carry-forward)    │
│    - nginx                  → reverse proxy fronting all of the above  │
│    - sqlite                 → rf_events.sqlite (thumb drive) +         │
│                               rickroll.db (SD card)                    │
└────────────────────────────────────────────────────────────────────────┘
```

### §1.4 — Role A: RF survey (always-on)

1. On boot, `udev` detects the SDR vendor/product IDs and triggers the systemd units.
2. `bmn-rf-scanner` claims SDR #2, loads scan ranges from `/opt/bmn-rf/config/scan-ranges.json`, and hops through frequencies at 2 s/dwell (~7 min per pass for ~200 freqs of interest — 6 passes/hour gives solid activity stats per the brief's "within an hour, know what's used here" goal).
3. When signal strength on a frequency exceeds threshold, scanner switches to FM demodulation, runs the audio through the CTCSS/DCS tone detector (Goertzel for CTCSS, Viterbi for DCS), and logs the transmission row to `rf_events.sqlite` on the thumb drive.
4. If `record_active = true` for that frequency in `channels.json` (or via a "promote to monitor" web-UI action), the scanner triggers `bmn-rf-recorder` to capture the WAV burst to `/opt/bmn-rf/recordings/YYYY-MM-DD/<filename>.wav` (filename per brief: `YYYY-MM-DD_HH-MM-SS_frequency_tone_or_CSQ_duration.wav`, America/Chicago local time).
5. Recording ends when carrier drops or duration cap reached. Audio path written back to the SQLite row.
6. OpenWebRX serves SDR #1 continuously for browser-based live listen. The web UI's "live now" tab links to the OpenWebRX waterfall.
7. Web UI on Pi `wlan0` shows dashboard, recordings browser, scan range editor, service health (CPU temp, disk space, SDR present, battery estimate via INA219 if added).

### §1.5 — Role B: DefCon captive portal Rickroll (toggleable)

When **DefCon Mode = OFF** (default):
- `wlan2` on the mAP lite is disabled.
- `/ip/hotspot` on the mAP lite is disabled.
- No captive portal SSIDs broadcast. Rig appears as just the mgmt network on `wlan0`.

When **DefCon Mode = ON** (the maintainer flips via web UI button OR tik shell script):
- The Pi web UI calls the tik REST API (restricted user, write scope = `/interface/wireless` + `/ip/hotspot`).
- Tik enables `wlan2` (2.4 GHz AP, multiple open SSIDs per v1 §5).
- Tik enables `/ip/hotspot` on the captive portal bridge ((internal IP)/24).
- Captive portal subnet is firewall-isolated: can ONLY reach Pi `eth0`:80 for the rickroll video (walled garden); blocked from tik admin, blocked from mgmt subnet, blocked from WAN, blocked from `wg0`.
- v1's portal app behavior (the Rickroll page, the social mirror, the metrics logging) is unchanged — that code lives in `/opt/bmn-rfrig/portal/` and binds to `eth0` on a dedicated port. v1's §7 (DNS hijack of CPD probes), §8 (social mirror), §10 (admin UI), and §13 (wifi-safety) all carry forward as-is.

Toggle audit: every flip writes syslog on both Pi and tik. the maintainer can read the trail in the web UI's DefCon-mode page.

### §1.6 — Why this Pi/tik split

- **mAP lite is the only thing that knows about wifi clients.** RouterOS HotSpot is mature, handles the captive-portal-detection protocol all phones expect, and does multi-SSID virtuals natively. We don't reimplement.
- **Pi is the only thing that does RF DSP, OpenWebRX, recordings, and the web UI.** It has CPU + USB bandwidth; the tik does not.
- **WireGuard is on the tik, not the Pi.** Two reasons: (a) the tik is the WAN-facing device, so terminating the tunnel there avoids Pi-side encrypt/decrypt overhead and keeps the Pi blissfully ignorant of when the tunnel is up; (b) BMN VPN hub's firewall is the single source of truth for "is this rig allowed to phone home right now?" — moving WG to the Pi would dilute that control.
- **DefCon toggle is tik-side because the radio + portal mechanics are tik-side.** The Pi button is just a remote control; the actual on/off lives where the wifi lives.

### §1.7 — Data flow during opportunistic NAS sync

Unattended horse-show / event-floor pattern:

1. Tik's `wlan1` (5 GHz STA) auto-associates with priority SSID list — Toaster (the maintainer's home wifi), phone hotspot, hotel wifi.
2. Tik gets WAN, NATs Pi via `ether2`.
3. Cron on **BMN VPN hub** opens BMN-side WG firewall during scheduled window (e.g. 03:00–04:00 nightly Central).
4. Tunnel negotiates from tik in ~25 s via PersistentKeepalive.
5. Pi `rsync`s `rf_events.sqlite` snapshot + recordings folder + portal `rickroll.db` snapshot to `operator NAS` ((internal IP), `/volume1/defcon-rickroll/` share) through the tik's tunnel.
6. Window closes at 04:00, tunnel dies until next window.
7. If rig out of range, sync resumes next viable window. Local data is never deleted by upload — the maintainer resets metrics manually via the admin UI.

A manual "open now" REST endpoint on workflow (or a `/admin/sync/open-window` button) → 30-min sliding window on BMN VPN hub for ad-hoc remote access. Same call works locally or over WG.

---

## §2 — Pi OS and package stack

### OS

**Raspberry Pi OS Lite, 64-bit, Bookworm** (carry-forward from v1 §2).

### Base packages (v1 carry-forward marked ✓; new for v2 marked ★)

| Package | Purpose | v1/v2 |
|---|---|---|
| `nginx` | Reverse proxy fronting OpenWebRX + scanner UI + portal app | ★ (v1 had Caddy; switching to nginx in v2 because OpenWebRX docs assume nginx and the per-vhost site-block ergonomics are similar enough not to fight) |
| `python3`, `python3-venv`, `python3-pip` | Flask app runtime, scanner service | ✓ |
| `gunicorn` (via venv) | WSGI server | ✓ |
| `sqlite3` | DB CLI + library | ✓ |
| `rsync`, `openssh-client` | NAS upload | ✓ |
| `iw`, `iproute2`, `wpasupplicant`, `hostapd`, `dnsmasq` | wlan0 mgmt AP | ★ |
| `chrony` | Time sync (matters for recording filenames + tone-detection timestamps) | ✓ |
| `unattended-upgrades` | Security updates | ✓ |
| `fail2ban` | SSH on wlan0 only | ✓ ((internal docs)) |
| `rtl-sdr` | librtlsdr + tools (`rtl_test`, `rtl_fm`, `rtl_power`, `rtl_tcp`) | ★ |
| `sox` | Audio post-processing for recordings | ★ |
| `lame` (optional) | MP3 transcode of WAV recordings for web-UI playback bandwidth savings | ★ |
| `openwebrx` (via official deb or Docker — Q-OPENWEBRX-INSTALL below) | Browser-based SDR receiver, multi-band, multi-demodulator | ★ |
| `usbutils`, `udev` | SDR detect + udev rules | ★ |
| `python3-numpy`, `python3-scipy` | DSP for the tone detector | ★ |

### Python deps (in a venv at `/opt/bmn-rfrig/venv`)

```
Flask>=3.0
gunicorn>=22.0
APScheduler>=3.10        # scanner hop loop, recording supervisor, sync triggers
SQLAlchemy>=2.0          # both DBs
Flask-Login>=0.6         # admin auth (carry-forward v1)
python-routeros>=0.7     # RouterOS API for DefCon toggle + lease/host data
requests>=2.31           # WAN probe + tik REST calls
pyrtlsdr>=0.2            # Python wrapper for librtlsdr (scanner side)
numpy>=1.26
scipy>=1.12              # signal processing for CTCSS Goertzel + DCS Viterbi
soundfile>=0.12          # WAV write
itsdangerous>=2.1        # session signing
```

### Filesystem layout

Umbrella `/opt/bmn-rfrig/` for the rig as a whole; the architect brief's `/opt/bmn-rf/` namespace preserved for the RF role's user-facing config and data; v1's `/var/lib/rickroll/` carries forward as `/var/lib/bmn-rfrig/portal/`.

```
/opt/bmn-rfrig/                          # umbrella for rig software
  venv/                                  # shared Python venv
  app/                                   # Flask app — both roles
    __init__.py
    portal/                              # v1 portal blueprint (carry-forward)
    rf/                                  # NEW: RF survey blueprint
      scanner.py                         # discovery scanner (SDR #2)
      monitor.py                         # locked monitor / recorder
      tone_detector.py                   # CTCSS/DCS detector
      api.py                             # /api/rf/* endpoints
      models.py                          # RF SQLAlchemy models
      templates/
      static/
    admin.py                             # unified admin UI (v1 + RF tabs)
    routeros.py                          # tik REST + API helpers (v1 + DefCon toggle)
    models.py                            # portal SQLAlchemy models (v1)
  scripts/
    upload_worker.py                     # v1 carry-forward + RF DB rows
    internet_probe.py                    # v1 carry-forward (tik calls now)
    seed_event.py                        # v1 carry-forward
    band_preset_seeder.py                # ★ load default scan ranges into JSON
  systemd/                               # see §1.3 services list
  nginx/                                 # vhost configs

/opt/bmn-rf/                             # the architect brief namespace — RF role
  config/
    scan-ranges.json                     # per brief
    channels.json                        # per brief — known channels + promoted-monitor list
  recordings/
    YYYY-MM-DD/                          # per brief — daily folders
      YYYY-MM-DD_HH-MM-SS_freq_tone_dur.wav
  logs/
    scanner.log
    monitor.log
    tone_detector.log
  db/
    rf_events.sqlite                     # per brief — RF event log
                                         # SYMLINK to thumb drive in production

/var/lib/bmn-rfrig/                      # stateful data
  portal/
    rickroll.db                          # v1 portal SQLite (SD card OK — low write rate)
    media/
      rickroll.mp4                       # v1 carry-forward
      poster.jpg
    social_mirror/                       # v1 carry-forward
    exports/                             # v1 carry-forward

/etc/bmn-rfrig/                          # config + secrets
  config.toml                            # event ID, NAS host, NAS path, admin password hash,
                                         # tik REST creds, OpenWebRX admin creds
  nas_known_hosts                        # SSH known_hosts pinned to operator NAS
  nas_id_ed25519                         # mode 0600, owned by rfrig user

/etc/systemd/system/
  bmn-rf-scanner.service                 # scanner (SDR #2)
  bmn-rf-monitor.service                 # locked-monitor recorder (SDR #2 promoted)
  bmn-rf-tone-detector.service           # standalone or in-process w/ scanner — TBD
  bmn-rf-webui.service                   # Flask app (gunicorn)
  bmn-portal.service                     # Flask app, eth0 bind (when DefCon Mode on)
  openwebrx.service                      # OpenWebRX (deb or Docker)
  bmn-rfrig-upload.service               # one-shot rsync run
  bmn-rfrig-upload.timer                 # opportunistic timer (WG-aware)
  hostapd.service / dnsmasq.service      # Pi wlan0 mgmt AP

/etc/nginx/sites-available/
  rfrig.conf                             # wlan0 vhost — admin, RF UI, OpenWebRX proxy
  portal.conf                            # eth0 vhost — rickroll only (when DefCon Mode on)

/mnt/usbdata/                            # thumb drive mount
  rf_events.sqlite                       # actual DB file (symlinked from /opt/bmn-rf/db/)
  recordings/                            # actual recordings (symlinked from /opt/bmn-rf/recordings/)
  openwebrx-logs/                        # OpenWebRX logs
```

### Service user

- Single non-login user `rfrig` (shell `/usr/sbin/nologin`).
- Owns `/opt/bmn-rfrig/`, `/opt/bmn-rf/`, `/var/lib/bmn-rfrig/`, `/etc/bmn-rfrig/`, `/mnt/usbdata/`.
- Member of `plugdev` group for raw USB access to SDRs (alternative: udev rule grants device-node access to `rfrig` group).
- Admin SSH on `wlan0` stays as the standard BMN `claude` admin user ((internal docs)). No service runs as root.

---

## §3 — Web/app framework + OpenWebRX

### Flask + Gunicorn + nginx (revised from v1's Caddy)

v1 used Caddy. v2 switches to **nginx** because:
- OpenWebRX's docs and community recipes assume nginx for reverse-proxy fronting (the WebSocket upgrade + audio MIME-type tuning is more battle-tested there).
- nginx's `server_name` + `listen 0.0.0.0:PORT` ergonomics give cleaner per-interface binding (we want admin / RF UI bound to `wlan0`, portal bound to `eth0`).
- We lose Caddy's automatic HTTPS — but the only HTTPS surface we cared about in v1 was the admin UI on the operator interface, and the rig is local-LAN-only (no public certs). Self-signed via `openssl` once, generated at build time, trusted by the maintainer's operator devices. Same end state.

Bind plan (enforced by nginx + nftables):

| Interface | Port | What's reachable | Auth |
|---|---|---|---|
| `wlan0` | 80 / 443 | Admin UI, RF UI, OpenWebRX (proxied at `/owrx/`), DefCon toggle, sync controls | HTTP Basic over HTTPS (self-signed cert) + nftables IP allowlist for mgmt subnet |
| `eth0` | 80 | **Only** the captive portal `/rickroll/`, `/social/*` mirror, `/api/log/*` — i.e. the walled-garden surface. Nothing else. | None — captive portal users are anonymous by design (v1 §7) |
| `wlan0` | 22 | SSH | Key-only ((internal docs)) |
| `wlan0` | 8073 | OpenWebRX direct (only for debugging — normally accessed via nginx `/owrx/`) | OpenWebRX's own user/pass |

### Why OpenWebRX

OpenWebRX is the centerpiece for the "view from my phone" requirement. It handles ~80% of what would otherwise be a from-scratch build: live waterfall, FM/AM/SSB/digital demodulation, multi-user browser sessions, mobile-friendly UI. Self-hosted (deb or Docker), no internet dependency. The bmn-rf-scanner + tone detector + web UI cover the other 20% (discovery, tone detection, recording, scan-range editing) — they're the "appliance" part of the appliance.

**Q-OPENWEBRX-INSTALL** in §14 covers deb vs Docker; default is Docker for isolation + easier upgrade story, fall back to deb if the Pi's USB→SDR latency profile suffers under Docker's overhead.

### Custom services (bmn-rf-*)

- **`bmn-rf-scanner`** (Python, APScheduler-driven): claims SDR #2, hops through frequencies from `scan-ranges.json` at configurable dwell time. Detects activity by signal level threshold. Writes `frequency_seen` rows to `rf_events.sqlite`. When a frequency goes "active" (signal sustained > N seconds), dispatches to the monitor.
- **`bmn-rf-monitor`** (Python): demodulates FM on the active frequency, pipes audio through the tone detector, and to `bmn-rf-recorder` (which writes the WAV burst). One transmission = one WAV file. Filename per brief.
- **`bmn-rf-tone-detector`** (Python): demodulates FM → low-pass filter below ~300 Hz → Goertzel detector for the 50 standard CTCSS tones (67–254 Hz) + Viterbi decoder for DCS digital codes. Returns `(tone | code | None, confidence)`. Result piped into the same `rf_events.sqlite` row.
- **`bmn-rf-webui`** (Flask via Gunicorn): scan range editor, discovered frequency table (hit count, last heard time, tone/code, promote-to-monitor button), recordings browser (play/download), service status. Bound to `wlan0` only.

Language: Python end-to-end for v2. **Q-SCAN-LANG** in §14 captures the "switch to Rust for scanner if Pi can't keep up" fallback.

---

## §4 — SQLite schemas

### §4.1 — Portal DB (carry-forward from v1 §4)

The full v1 portal schema (`events`, `devices`, `portal_hits`, `video_events`, `link_clicks`, `uploads`, `health`) is preserved verbatim. See `00_DESIGN_v1.md` §4. Lives at `/var/lib/bmn-rfrig/portal/rickroll.db` (SD card — low write rate, WAL mode).

### §4.2 — RF events DB (NEW for v2, on thumb drive)

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

-- A scan campaign — typically reset per deployment/event.
CREATE TABLE rf_events (
    rf_event_id   TEXT PRIMARY KEY,         -- e.g. 'rfe-2026-08-09-defcon-sat-pm'
    label         TEXT NOT NULL,
    started_at    TEXT NOT NULL,            -- ISO-8601 UTC, scanner write
    ended_at      TEXT,
    notes         TEXT
);

-- Scan range definitions snapshot at event start.
-- The live editable copy is /opt/bmn-rf/config/scan-ranges.json;
-- this is the immutable record of what was scanned during the event.
CREATE TABLE scan_ranges (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    rf_event_id   TEXT NOT NULL REFERENCES rf_events(rf_event_id) ON DELETE CASCADE,
    name          TEXT NOT NULL,            -- 'GMRS', 'Ham 2m', 'Business VHF'
    start_hz      INTEGER NOT NULL,
    stop_hz       INTEGER NOT NULL,
    step_hz       INTEGER NOT NULL,
    dwell_ms      INTEGER NOT NULL,
    modulation    TEXT NOT NULL,            -- 'NFM','WFM','AM' default for the range
    enabled       INTEGER NOT NULL DEFAULT 1
);

-- A scan pass — one full sweep through all ranges.
CREATE TABLE scan_passes (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    rf_event_id   TEXT NOT NULL REFERENCES rf_events(rf_event_id) ON DELETE CASCADE,
    started_at    TEXT NOT NULL,
    finished_at   TEXT,
    sdr_serial    TEXT NOT NULL,            -- which SDR did the pass
    pass_number   INTEGER NOT NULL,
    freq_count    INTEGER,
    hit_count     INTEGER                   -- frequencies that exceeded threshold
);

-- A frequency observed at a moment.
CREATE TABLE frequencies_observed (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    rf_event_id   TEXT NOT NULL REFERENCES rf_events(rf_event_id) ON DELETE CASCADE,
    scan_pass_id  INTEGER REFERENCES scan_passes(id),
    seen_at       TEXT NOT NULL,
    freq_hz       INTEGER NOT NULL,
    signal_dbm    REAL,
    modulation    TEXT,
    sdr_serial    TEXT NOT NULL
);

-- A discrete transmission — carrier-up to carrier-down.
CREATE TABLE transmissions (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    rf_event_id   TEXT NOT NULL REFERENCES rf_events(rf_event_id) ON DELETE CASCADE,
    freq_hz       INTEGER NOT NULL,
    started_at    TEXT NOT NULL,
    ended_at      TEXT,
    duration_s    REAL,
    peak_dbm      REAL,
    mean_dbm      REAL,
    modulation    TEXT NOT NULL,
    sdr_serial    TEXT NOT NULL,
    audio_path    TEXT,                     -- relative to /opt/bmn-rf/recordings/
    band_label    TEXT,                     -- 'GMRS', 'Ham 2m', etc. (from scan_ranges.name)
    encryption_observed INTEGER,             -- 0 = clear, 1 = digital-but-not-decoded, 2 = encrypted+confirmed
    notes         TEXT
);

-- Squelch tone detections. One row per detection; multiple detections per transmission possible
-- if the tone changes (rare in practice — tone is usually stable through a TX).
CREATE TABLE tone_detections (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    transmission_id INTEGER NOT NULL REFERENCES transmissions(id) ON DELETE CASCADE,
    detected_at   TEXT NOT NULL,
    tone_type     TEXT NOT NULL CHECK (tone_type IN ('CTCSS','DCS','NAC','ColorCode','TalkgroupID','None')),
    tone_value    TEXT NOT NULL,            -- '127.3' for CTCSS Hz, '023' for DCS, '$293' for NAC, etc.
    confidence    REAL                       -- 0..1
);

-- Known channels / named frequencies (loaded from /opt/bmn-rf/config/channels.json on boot).
-- Used by the web UI to enrich raw freq numbers with human-readable labels.
CREATE TABLE known_channels (
    freq_hz       INTEGER PRIMARY KEY,
    label         TEXT NOT NULL,            -- 'GMRS 1 (462.5625 MHz)'
    band_label    TEXT NOT NULL,            -- 'GMRS'
    expected_tone TEXT,                     -- if known: '127.3' / 'DCS 023' / NULL
    record_active INTEGER NOT NULL DEFAULT 0,-- promote to monitor by default?
    notes         TEXT
);

-- SDR health snapshots — periodic, for the Service Status UI.
CREATE TABLE sdr_health (
    seen_at       TEXT PRIMARY KEY,
    sdr1_present  INTEGER,
    sdr2_present  INTEGER,
    cpu_pct       REAL,
    cpu_temp_c    REAL,
    disk_free_mb  INTEGER,                  -- /mnt/usbdata
    rms_load      REAL,
    battery_pct   INTEGER,                  -- if INA219 wired; NULL otherwise
    runtime_est_h REAL                      -- if battery_pct present
);

-- Indexes
CREATE INDEX transmissions_event_started ON transmissions(rf_event_id, started_at DESC);
CREATE INDEX transmissions_freq           ON transmissions(rf_event_id, freq_hz);
CREATE INDEX freq_observed_event_seen     ON frequencies_observed(rf_event_id, seen_at DESC);
CREATE INDEX tone_det_transmission        ON tone_detections(transmission_id);
```

### §4.3 — Database placement

- `rickroll.db` → `/var/lib/bmn-rfrig/portal/rickroll.db` (SD card, low churn).
- `rf_events.sqlite` → physical file on `/mnt/usbdata/rf_events.sqlite`, symlinked at `/opt/bmn-rf/db/rf_events.sqlite` (matches brief's path; keeps the brief's documented API stable).
- `recordings/` → physical at `/mnt/usbdata/recordings/`, symlinked at `/opt/bmn-rf/recordings/`.

Both DBs run in WAL mode. **Q-RF-DB** in §14 covers the "merge into one DB" alternative; default is keep separate (cleaner failure isolation; RF role can survive portal DB corruption and vice versa).

---

## §5 — MikroTik dual-role configuration pattern

Target: RouterOS 7.x stable. Skeleton — paste-ready RSC will be generated as part of Phase 2.

### §5.1 — Bridges and subnets

```
/interface bridge
add name=bridge-mgmt    comment="mgmt subnet — Pi + the maintainer's devices"
add name=bridge-captive comment="captive portal subnet — DefCon Mode only"

/interface bridge port
add bridge=bridge-mgmt    interface=ether2
add bridge=bridge-captive interface=wlan2          ;# multiple virtuals also attached

/ip address
add address=(internal IP)/24 interface=bridge-mgmt    comment="mgmt gw"
add address=(internal IP)/24 interface=bridge-captive comment="captive portal gw"
```

### §5.2 — wlan1 (5 GHz STA) — WAN uplink, priority SSID list

```
/interface wireless security-profiles
add name=wpa2-toaster mode=dynamic-keys authentication-types=wpa2-psk \
    wpa2-pre-shared-key="<from-_credentials/wifi-toaster.md>"
# additional profiles per priority SSID — phone hotspot, hotel, etc.

/interface wireless
set [find default-name=wlan1] \
    band=5ghz-a/n/ac mode=station-bridge \
    ssid="" security-profile=wpa2-toaster disabled=no \
    wireless-protocol=802.11

# priority list — RouterOS scans these in order until one associates
/interface wireless connect-list
add interface=wlan1 ssid=Toaster              security-profile=wpa2-toaster
add interface=wlan1 ssid=AaronPhoneHotspot    security-profile=wpa2-aaron-phone
add interface=wlan1 ssid=HotelGuest_xyz       security-profile=open
```

### §5.3 — wlan2 (2.4 GHz AP, DefCon Mode) — DISABLED BY DEFAULT

Carry-forward from v1 §5 wireless block; key change is `disabled=yes`:

```
/interface wireless security-profiles
add name=open authentication-types="" mode=none unicast-ciphers="" group-ciphers=""

/interface wireless
set [find default-name=wlan2] \
    band=2ghz-b/g/n channel-width=20mhz frequency=auto \
    mode=ap-bridge ssid="Free WiFi - Lobby" \
    security-profile=open disabled=yes \                 ;# DEFAULT OFF
    wmm-support=enabled \
    wireless-protocol=802.11

add master-interface=wlan2 ssid="Conference WiFi" name=wlan2-virt1 \
    security-profile=open disabled=yes
add master-interface=wlan2 ssid="Hotel Guest"     name=wlan2-virt2 \
    security-profile=open disabled=yes
add master-interface=wlan2 ssid="Coffee Shop Free WiFi" name=wlan2-virt3 \
    security-profile=open disabled=yes
```

> SSID names are clearly generic (v1 §13 guardrail). the maintainer picks venue-appropriate-but-not-deceptive list per event.

### §5.4 — DHCP/DNS for each subnet

```
/ip pool
add name=mgmt-pool    ranges=(internal IP)-(internal IP)
add name=captive-pool ranges=(internal IP)-(internal IP)

/ip dhcp-server
add interface=bridge-mgmt    address-pool=mgmt-pool    name=mgmt-dhcp    lease-time=1h  disabled=no
add interface=bridge-captive address-pool=captive-pool name=captive-dhcp lease-time=15m disabled=yes

/ip dhcp-server network
add address=(internal IP)/24 gateway=(internal IP) dns-server=(internal IP)
add address=(internal IP)/24 gateway=(internal IP) dns-server=(internal IP)
```

The captive DHCP server is `disabled=yes` by default; the DefCon toggle flips it on with wlan2 and the hotspot.

### §5.5 — HotSpot (captive portal — DefCon Mode only)

```
/ip hotspot profile
add name=defcon login-by=http-pap http-cookie-lifetime=1h \
    use-radius=no html-directory=hotspot dns-name=portal.local \
    smtp-server=0.0.0.0 split-user-domain=no

/ip hotspot
add name=defcon interface=bridge-captive address-pool=captive-pool profile=defcon \
    addresses-per-mac=2 disabled=yes                    ;# DEFAULT OFF

/ip hotspot walled-garden
add dst-host=(internal IP) action=allow                 ;# Pi mgmt-side address
add dst-host=portal.local  action=allow
add dst-host=*.bytemenetworks.com action=allow

/ip hotspot walled-garden ip
add dst-address=(internal IP) action=accept             ;# captive→Pi:80 rickroll only
add dst-address=(internal IP)/24 action=accept
```

Caveat: captive portal subnet allows **only** traffic to the Pi's `eth0`:80 rickroll endpoint. Pi `wlan0` (admin / RF UI) is on a different bridge + subnet and is unreachable from the captive portal users. Per §1.5.

### §5.6 — Firewall (segment isolation — §13b critical)

```
/ip firewall filter
# captive portal users may ONLY reach Pi mgmt-side for the rickroll endpoint
add chain=forward in-interface=bridge-captive dst-address=(internal IP) dst-port=80 protocol=tcp action=accept \
    comment="captive → Pi rickroll"
add chain=forward in-interface=bridge-captive out-interface=bridge-mgmt action=drop \
    comment="captive → mgmt (DROP)"
add chain=forward in-interface=bridge-captive out-interface=wg0 action=drop \
    comment="captive → wg (DROP)"
add chain=forward in-interface=bridge-captive out-interface=wlan1 action=drop \
    comment="captive → WAN (DROP — walled garden)"
add chain=input   in-interface=bridge-captive dst-port=22,80,443,8291 protocol=tcp action=drop \
    comment="captive → tik admin (DROP)"
add chain=forward in-interface=bridge-captive out-interface=bridge-captive action=drop \
    comment="client isolation"
# mgmt → anywhere allowed by default
```

### §5.7 — WireGuard client to BMN VPN hub

```
/interface wireguard
add name=wg0 listen-port=51820 private-key="<from-_credentials/wg-bmn-rfrig.md>"

/interface wireguard peers
add interface=wg0 public-key="<terra-prime-public>" \
    endpoint-address=<terra-prime-public-ip> endpoint-port=51820 \
    allowed-address=(internal IP)/16,172.16.0.0/12 \                ;# BMN home only, NOT 0.0.0.0/0
    persistent-keepalive=25s

/ip route
add dst-address=(internal IP)/16 gateway=wg0
add dst-address=(internal IP)/24 gateway=wg0                        ;# Valley Mills DC
add dst-address=(internal IP)/24 gateway=wg0                        ;# operator site home
```

**Q-WG-ENDPOINT** in §14 covers the exact AllowedIPs scope and which BMN VPN hub side firewall rule opens the listening window.

### §5.8 — DefCon toggle: shell scripts + REST API user

Shell scripts on the tik for hardwire-in fallback:

```
# /etc/scripts/enable-defcon.sh (RouterOS scripting)
/interface wireless enable wlan2
/interface wireless enable wlan2-virt1
/interface wireless enable wlan2-virt2
/interface wireless enable wlan2-virt3
/ip hotspot enable defcon
/ip dhcp-server enable captive-dhcp
:log info "DEFCON_MODE_ENABLED by script"

# /etc/scripts/disable-defcon.sh
/interface wireless disable wlan2
/interface wireless disable wlan2-virt1
/interface wireless disable wlan2-virt2
/interface wireless disable wlan2-virt3
/ip hotspot disable defcon
/ip dhcp-server disable captive-dhcp
:log info "DEFCON_MODE_DISABLED by script"
```

Restricted RouterOS REST API user for the Pi web-UI button (per the maintainer's defense-in-depth requirement):

```
/user group add name=defcon-toggle policy=api,rest-api,read,!write
# (we use the per-path REST policy ACL to grant write on JUST these paths)
/user add name=pi-defcon-toggle group=defcon-toggle password="<from-_credentials>"

# RouterOS 7.x REST API access control
/console-color
/access add path=/interface/wireless permission=write user=pi-defcon-toggle
/access add path=/ip/hotspot         permission=write user=pi-defcon-toggle
/access add path=/ip/dhcp-server     permission=write user=pi-defcon-toggle
```

(Exact ACL syntax verified at Phase 2 build time — RouterOS REST ACL semantics vary between minor versions.)

If the Pi is compromised, the attacker can only flip the DefCon portal on/off — no pivot to tik internals, no access to wg0, no WAN reconfigure.

---

## §6 — Pi ↔ tik data flow

Carry-forward from v1 §6 (three channels: syslog over UDP, RouterOS API pull for lease/host data, captive portal HTTP POSTs with `?mac=` query string) — applies to the DefCon Mode operation unchanged.

**v2 additions:**

### §6.1 — REST API for the DefCon toggle (Pi → tik)

Pi web UI → POST to `https://(internal IP)/rest/interface/wireless/{numbers}/enable` (and friends) using HTTP Basic with `pi-defcon-toggle` creds. Wrapped in a small Python helper `app/routeros.py:toggle_defcon_mode(on: bool)`. The helper:
- Verifies cert against pinned thumbprint (the tik's self-signed cert; pinned at build time in `/etc/bmn-rfrig/tik_cert_fingerprint`).
- Logs the toggle attempt to syslog AND to `rickroll.db` `defcon_toggle_log` table (new).
- Returns success/failure for the UI to render.

### §6.2 — STA association status + WAN reachability (tik → Pi)

The internet probe service in v1 §1 polled from the Pi outbound. v2 inverts: Pi reads tik state via RouterOS API:
- `/interface/wireless/registration-table` on wlan1 STA → which SSID associated, signal, since when.
- `/interface/wireguard/peers` on wg0 → handshake recency (= "is BMN VPN hub firewall open").
- `/ip/route/check 8.8.8.8` → end-to-end WAN reachability.

These surface as the "Connectivity" tile on the admin UI Health page.

---

## §7 — Captive portal behavior

**Unchanged from v1 §7.** All CPD probe URL rewrites, `portal.local` hostname, captive portal HTML/CSS/JS, autoplay-muted → click-to-unmute UX, "Exit Wi-Fi" button, transparency page link from footer — all carry forward verbatim. The only difference is the entire portal stack is dormant when DefCon Mode is off; the systemd unit `bmn-portal.service` is `WantedBy=defcon-mode.target` rather than `multi-user.target`, and the target itself is enabled/disabled by the toggle helper.

---

## §8 — `social.bytemenetworks.com` mirror

**Unchanged from v1 §8.** Hand-built static fallback page with QR code, served from `/var/lib/bmn-rfrig/portal/social_mirror/`, DNS rewrite chain on the tik (`*.bytemenetworks.com → (internal IP)` when DefCon Mode is on; not rewritten when off). HTTP-only — no fake TLS for domains we don't own (§13 hard NO).

---

## §9 — NAS upload + WG sync strategy

### §9.1 — Mechanics (carry-forward from v1 §9, plus RF data)

Carry-forward: rsync over SSH with key auth; snapshot-first; idempotent (`--partial --append-verify`); upload ledger; no `--delete`; `command=` restriction on the NAS side.

**v2 addition:** the upload payload now includes both `rickroll.db` AND `rf_events.sqlite` AND the recordings tree. Per-event folder structure on the NAS:

```
/volume1/defcon-rickroll/
  evt-2026-08-09-defcon-sat-pm/
    portal/
      rickroll-evt-2026-08-09-defcon-sat-pm-<utc>.db
      devices.csv portal_hits.csv video_events.csv link_clicks.csv
      uploads.csv
    rf/
      rf_events.sqlite             # snapshot
      scan_ranges.csv frequencies_observed.csv transmissions.csv tone_detections.csv
      recordings/                  # rsync'd from /mnt/usbdata/recordings/
        YYYY-MM-DD/
          *.wav
    metadata.json
```

NAS target = `operator NAS` ((internal IP)) per the maintainer's 2026-04-28 decision. The `/volume1/defcon-rickroll/` share creation + `rickroll` user setup is still on the `[ThinkStation]` backlog.

### §9.2 — Trigger model (revised)

v1 had the Pi probe internet directly. v2 inverts because WAN now lives on the tik and WG-tunnel state is the canonical "is BMN reachable" signal:

1. **Tik-side keepalive** — every 30 s, the Pi reads `/interface/wireguard/peers wg0 last-handshake` from the tik API.
2. If last-handshake < 60 s → tunnel is up → BMN reachable → eligible for upload.
3. If last-handshake > 60 s → tunnel is closed (BMN VPN hub firewall not open) → skip.
4. Cooldown since last successful upload > N min (default 10) → run.
5. **Manual "open now" path:** the maintainer hits `/admin/sync/open-window` on the Pi UI, which posts a request to a Terra-Prime-side endpoint to open the firewall for a 30-min sliding window. Same Pi-side trigger logic then runs.

The scheduled BMN VPN hub cron is the unattended path; the manual button is the ad-hoc path.

### §9.3 — Snapshot ordering

1. `sqlite3 /var/lib/bmn-rfrig/portal/rickroll.db ".backup /var/lib/bmn-rfrig/portal/exports/rickroll-<ts>.db"`
2. `sqlite3 /opt/bmn-rf/db/rf_events.sqlite ".backup /mnt/usbdata/exports/rf_events-<ts>.sqlite"`
3. Write CSVs of all tables alongside.
4. `rsync` the snapshot dirs + the recordings dir to NAS over WG.
5. Update `uploads` row with status + sha256.

---

## §10 — Admin UI layout (v1 + RF tabs)

Carry-forward from v1 §10 dashboard, recent connections, vendor breakdown — for the portal role. New tabs added for the RF role + the rig overall:

```
┌──────────────────────────────────────────────────────────────────────┐
│  BMN RF Survey Rig — Admin              [🟢 RF] [🟢 NAS] [⚪ DefCon]  │
├──────────┬───────────────────────────────────────────────────────────┤
│          │                                                           │
│  🛰 RF   │  RF SECTION                                               │
│  📡 Live │   - Dashboard (active freq, last heard, CTCSS, level)     │
│  📜 Scan │   - Discovered frequency table (hit count, last, tone,    │
│  📁 Recs │     promote-to-monitor button)                            │
│  ⚙ Cfg   │   - Recordings browser (play/download)                    │
│          │   - Scan range editor                                     │
│  🎭 DefC │                                                           │
│  📊 Port │  DEFCON / PORTAL SECTION (v1 carry-forward)               │
│  📡 Live │   - Dashboard, recent connections, vendor breakdown       │
│  📜 Hist │   - Live tail, events, devices, export                    │
│  ⬇ Exp   │   - NAS, Reset                                            │
│  ☁ NAS   │                                                           │
│          │  SHARED                                                   │
│  ⚙ Hlth  │   - DefCon toggle button (+ "this will start              │
│  🔄 Sync │     broadcasting" warning + 2-click confirm)              │
│  🔄 Rset │   - Sync to BMN ("Open WG window" button + history)       │
│          │   - Health: SDR present, CPU temp, disk free,             │
│          │     battery est, WAN/STA, WG state                        │
│          │   - Service control: restart scanner / restart            │
│          │     monitor / reload config                               │
└──────────┴───────────────────────────────────────────────────────────┘
```

### Pages (full set)

| Path | Purpose |
|---|---|
| `/admin/` | Rig overview — RF active count, DefCon state, NAS reachability, last sync |
| `/admin/rf/dashboard` | RF active freq, last heard, CTCSS/DCS, signal level, duration, recording link |
| `/admin/rf/scan-ranges` | Edit `scan-ranges.json` (form + JSON view + per-band toggle) |
| `/admin/rf/discovered` | Discovered freq table — hit count, last heard, tone/code, promote-to-monitor button |
| `/admin/rf/recordings` | Browser — by date, by freq, by band; play (HTML5 audio), download (WAV/MP3) |
| `/admin/rf/live` | Embedded OpenWebRX iframe (proxied through nginx `/owrx/`) |
| `/admin/rf/config` | Edit `channels.json` (known channels w/ named labels) |
| `/admin/defcon/state` | DefCon mode current state, toggle button (2-click confirm + log) |
| `/admin/portal/*` | v1 §10 pages — dashboard, live, hist, devices, export, NAS, reset (only meaningful when DefCon Mode has been on at some point) |
| `/admin/health` | Service status — SDR present, CPU temp, disk free `/mnt/usbdata`, battery est, WAN STA SSID + signal, WG handshake age, all systemd unit states |
| `/admin/sync` | NAS sync history + "Open WG window now" button |
| `/admin/services` | Restart scanner / monitor / portal / openwebrx; reload configs |

### Auth (carry-forward v1 §10)

HTTP Basic over HTTPS (self-signed cert), bcrypt-hashed creds in `config.toml`, IP allowlist enforced by nftables on `wlan0` (mgmt subnet only), Flask-Login session w/ strict cookies. **Q-AUTH** in §14 still open: TOTP factor?

---

## §11 — Battery / runtime (revised for dual-role draw)

### Revised power draw

| Device | Idle (W) | Typical (W) | Peak (W) |
|---|---|---|---|
| Raspberry Pi 4B (no peripherals) | ~3.0 | ~5.0 | ~7.5 |
| MikroTik mAP lite | ~1.5 | ~2.0 | ~3.0 |
| 2 × NooElec RTL-SDR (each ~0.7–1.0 W) | ~1.4 | ~2.0 | — |
| USB thumb drive (read/write) | ~0.2 | ~0.5 | ~1.0 |
| **Combined typical** | **~6.1** | **~9.5** | **~12.5+** |

### Revised battery sizing

| Bank label | Realistic Wh @ 5V | Runtime @ 9.5 W typical |
|---|---|---|
| 20,000 mAh | ~56 Wh | ~5.5 h |
| 26,800 mAh | ~75 Wh | ~7.5 h |
| 40,000 mAh | ~110 Wh | ~11 h |
| Anker PowerHouse 250 Wh class | ~200 Wh usable | ~20 h |

**the maintainer's existing Anker PowerHouse-equivalent** (decision in synopsis) gets a comfortable overnight run with margin. The smaller pocket banks (10–20 kAh) are for short DefCon-floor stints.

Other v1 §11 details (USB power monitor, low-power-mode flag, pass-through charging note, FAA carry-on note, field test cadence) carry forward unchanged.

---

## §12 — Phased build plan (revised)

Each phase ends in a checkpoint. Phases sized to fit one (task queue) for the maintainer ((per internal convention)).

### Phase 0 — Prereqs & decisions (no code)

- License confirmed (Community Use, interim placeholder — canonical CU pending; see §14 Q-LICENSE).
- NAS target decided (operator NAS, `/volume1/defcon-rickroll/`); share + `rickroll` user setup queued (see GLOBAL_BACKLOG `[ThinkStation]` item).
- Battery confirmed (the maintainer's Anker for overnight; pocket banks for DefCon floor).
- mAP lite onboarded (per internal convention) — admin password set, REST API access enabled, RouterOS 7.x stable.
- 2 × NooElec SDRs + thumb drive in hand. SDR vendor/product IDs captured for udev.
- Antennas in hand per Option B (two dedicated) — discone for survey, targeted whip for monitor.
- Local Rick Astley MP4 sourced legally (v1 carry-forward; the maintainer supplies).
- Win10 laptop RF-capture playbook started at `05_LAPTOP_PLAYBOOK/` (parallel workstream — informs decisions about modulation defaults, tone-detector tuning).

### Phase 1 — Pi base image

- Raspberry Pi OS Lite 64-bit installed.
- BMN standard SSH banner ((internal docs)); BMN authorized_keys sync configured ((internal docs)).
- `rfrig` service user; `/opt/bmn-rfrig/`, `/opt/bmn-rf/`, `/var/lib/bmn-rfrig/`, `/etc/bmn-rfrig/` directories.
- nginx + Python venv installed.
- Thumb drive mounted at `/mnt/usbdata/` (UUID in fstab; mounted before bmn-rf-* units start).
- Hostname `bmn-rfrig-pi`.

**Checkpoint:** `ssh claude@bmn-rfrig-pi` works (mgmt path via tik); nginx serves a "hello, ByteMe" page on `wlan0`:80.

### Phase 2 — RouterOS dual-role base

- mAP lite reset.
- Bridges + subnets configured per §5.1.
- wlan1 STA mode + priority SSID list per §5.2 — confirm WAN uplink works.
- wlan2 + virtuals defined per §5.3 — DEFAULT DISABLED.
- DHCP + DNS per §5.4 (captive DHCP disabled).
- HotSpot + walled garden defined per §5.5, disabled.
- Firewall per §5.6 — segment isolation rules in place even though wlan2 is off (defense in depth).
- WireGuard client per §5.7 — peer config, but BMN VPN hub side firewall NOT yet opened (verify dormant).
- DefCon toggle scripts per §5.8 — `enable-defcon.sh` / `disable-defcon.sh` on tik.
- Pi mgmt AP on `wlan0` brought up (hostapd + dnsmasq).

**Checkpoint:** Pi reachable from the maintainer's phone via Pi `wlan0` mgmt SSID; Pi `eth0` reaches internet via tik; `wlan2` is silent (no SSIDs broadcast); `enable-defcon.sh` brings up wlan2 + portal redirect to Pi placeholder page; `disable-defcon.sh` takes it back down.

### Phase 3 — OpenWebRX + SDR #1 live audio

- Install OpenWebRX (deb or Docker — Q-OPENWEBRX-INSTALL).
- Configure SDR #1 as primary receiver (band plan: Ham/GMRS/Business defaults).
- Bind OpenWebRX to localhost; reverse-proxy via nginx `/owrx/` on `wlan0`:80.
- Confirm browser playback from the maintainer's phone on mgmt SSID.

**Checkpoint:** the maintainer opens phone browser → http://bmn-rfrig-pi.local/owrx/ → live waterfall + audio works.

### Phase 4 — Pi web UI shell + DefCon toggle wiring

- Flask app skeleton with `/admin/`, `/admin/health`, `/admin/defcon/state`.
- HTTPS via self-signed cert; HTTP Basic auth; IP allowlist via nftables.
- `routeros.py` helper with `toggle_defcon_mode(on: bool)` — REST call to tik via restricted user.
- 2-click confirm UX before the toggle fires; syslog + DB log.

**Checkpoint:** the maintainer logs into admin UI, clicks DefCon ON, confirms — wlan2 lights up, captive portal SSIDs broadcast, joining client lands on placeholder. Toggles OFF; wlan2 silent again.

### Phase 5 — RF scanner service (SDR #2)

- `bmn-rf-scanner` service per §1.4.
- `scan-ranges.json` seeded with default presets (Ham, MURS, GMRS/FRS, NOAA, Business, Airband RX, marine VHF RX).
- `rf_events.sqlite` schema deployed (§4.2).
- Scanner hops, detects, logs `frequencies_observed` rows.

**Checkpoint:** scanner runs continuously; admin UI Discovered Frequencies table shows hits when test radio TXes on a known frequency.

### Phase 6 — Monitor/recorder + CTCSS/DCS tone detector

- `bmn-rf-monitor` triggered when scanner sees sustained activity.
- `bmn-rf-tone-detector` integrated into the monitor pipeline.
- Recordings written to `/mnt/usbdata/recordings/YYYY-MM-DD/` with brief-spec'd filename.
- `transmissions` + `tone_detections` rows populated.

**Checkpoint:** transmit a CTCSS-encoded GMRS test signal; recording lands on thumb drive with correct filename, tone correctly identified in DB.

### Phase 7 — RF web UI MVP

- `/admin/rf/dashboard` (live RF state).
- `/admin/rf/discovered` (frequency table with promote-to-monitor button).
- `/admin/rf/recordings` (browser w/ play/download).
- `/admin/rf/scan-ranges` (editor — writes back to JSON file).
- `/admin/rf/config` (channels.json editor).

**Checkpoint:** the maintainer drives full RF UI from phone on mgmt SSID.

### Phase 8 — Captive portal carry-forward (v1 §3 Phase 3)

- v1 portal app `/opt/bmn-rfrig/portal/` lifted in; binds to `eth0`:80 via nginx vhost; activates only when `defcon-mode.target` is up.
- v1 SQLite schema (`rickroll.db`) deployed.
- Local Rickroll video in place; portal HTML/CSS/JS shipped.

**Checkpoint:** with DefCon Mode ON, joining client gets Rickrolled; hit appears in portal admin tab.

### Phase 9 — social.bytemenetworks.com mirror

Carry-forward from v1 §3 Phase 4. DNS rewrite on tik for `*.bytemenetworks.com → (internal IP)` enabled with DefCon Mode; nginx vhost on Pi `eth0` for the host; static fallback page + QR.

**Checkpoint:** in DefCon Mode, tap link on portal page → land on Pi mirror; QR scan goes to real URL when off the prank network.

### Phase 10 — Pi nftables hardening

- nftables rules per §3 bind plan: only `/rickroll/*`, `/social/*`, `/api/log/*` on `eth0`:80; everything else dropped.
- `wlan0` listening for admin/RF UI/SSH/OpenWebRX only.
- Verify captive portal users cannot probe the RF UI or OpenWebRX (port scan from captive subnet should yield only the rickroll endpoint).

**Checkpoint:** from a captive-portal-joined device, `curl http://(internal IP)/admin/` → connection refused.

### Phase 11 — WireGuard sync window pattern

- BMN VPN hub side: cron opens BMN firewall for wg0 peer at scheduled window (e.g. 03:00–04:00 Central nightly).
- Manual "open now" endpoint on BMN VPN hub — 30-min sliding window — called by Pi admin UI button.
- Pi sync trigger logic per §9.2 (read tik wg0 last-handshake; if fresh → upload).
- rsync over WG to `operator NAS:/volume1/defcon-rickroll/<event>/`.

**Checkpoint:** at scheduled window with rig powered on inside SSID range, upload lands on operator NAS; manual "open now" triggers ad-hoc sync.

### Phase 12 — Multi-SSID hardening + venue-list lock for DefCon Mode

- the maintainer picks the venue-appropriate-but-not-deceptive SSID list per event.
- Multi-SSID virtuals confirmed all land on the same portal flow (when DefCon Mode on).
- Client isolation on captive bridge confirmed (client A cannot ping client B).

**Checkpoint:** all SSIDs land on the portal; clients isolated; tik admin unreachable from captive subnet.

### Phase 13 — Battery + portability + field test

- Bench test on Anker battery for ≥ 4 hours under simulated load (scanner running, OpenWebRX serving 1 browser, no DefCon Mode).
- Carry case / strap as the maintainer prefers.
- "Off switch" procedure documented (RouterOS `/system shutdown`, Pi `sudo halt`).
- Dry run at home — confirm sync lands on operator NAS; confirm DefCon toggle works from phone.

**Checkpoint:** the maintainer walks rig out the door, confirms it works, comes back, checks upload landed.

### Phase 14 — Polish (post first event)

- Dashboard improvements based on what was actually useful.
- Logging cleanup.
- Documentation pass for repo / runbook.
- Promotion of `05_LAPTOP_PLAYBOOK/` to sibling project per §14 Q-LAPTOP-PROMOTE if it has earned its own folder by now.

---

## §13 — Security / safety boundaries (wifi-side — carry-forward from v1)

**Unchanged from v1 §13** — Hard NO list (no credential capture, no phishing, no traffic interception, no TLS MITM, no malware delivery, no payment forms, no content-of-traffic logging, no persistent cross-event tracking surfaced in UI) and Hard YES list (clear "this is a prank" disclosure, client isolation on, admin reachable only from operator interfaces, all admin auth over HTTPS, all keys in `_credentials/`, off switch is obvious, local-only-by-default) all apply to the DefCon Mode operation.

These boundaries are non-negotiable. Read `00_DESIGN_v1.md` §13 in full before touching anything that could drift toward credential capture, traffic interception, or trusted-SSID impersonation.

---

## §13b — Security / safety boundaries (RF survey side — NEW for v2)

These are non-negotiable. They cover the always-on RF role. They are written tighter than §13 because RF monitoring sits in a legally-cleaner-but-publicly-touchier space than a captive-portal prank.

### §13b.1 — The legal foundation

US federal law on monitoring radio:

- **47 USC §605 (Communications Act):** Prohibits *divulging or publishing* certain intercepted communications without the sender's consent. **Does not prohibit reception** of unencrypted radio communications. Specifically carves out broadcast stations and radio communications "for the use of the general public."
- **18 USC §2511 (ECPA / Wiretap Act):** Prohibits intentional interception of wire/oral/electronic communications. **Specifically excludes** "any radio communication which is transmitted ... by any governmental, law enforcement, civil defense, private land mobile, or public safety communications system, including police and fire, readily accessible to the general public." "Readily accessible to the general public" = not scrambled, not encrypted, not on a cellular frequency, not using modulation techniques withheld from the public.
- **47 USC §605(a):** Specifically prohibits divulging the contents of intercepted **cellular** communications. (Cell phone interception is excluded from the rig's scope — the SDRs can't reach LTE/5G with usable demod anyway, and we deliberately exclude any cellular bands from `scan-ranges.json` defaults.)

**the maintainer's stated posture (2026-05-25):** "I am not attempting to decode government encrypted frequencies, because of this listening in is considered legal at all aspects, included decoding any modulation etc. So I want all available data, including privacy tones etc. as there is nothing that legally protects those."

That maps cleanly onto the legal framework above: monitor anything unencrypted that the SDR can hear; do not attempt to break encryption on any system; cellular is out of scope; tones (CTCSS/DCS) and other unencrypted control data (talkgroup IDs, NAC, color codes, station IDs) are fair game because they are not legally protected.

### §13b.2 — Hard NO list (RF role)

1. **No attempt to break encryption.** If a transmission has encryption set (P25 ENC bit, DMR encryption flag, Hytera AES-256, etc.), the rig logs **metadata only** — frequency, modulation, time, encryption-present flag (`encryption_observed = 2` in `transmissions`) — and stops decoding the audio payload. No key recovery attempts. No brute-force. No exploit of vendor-specific weak-cipher implementations. This applies *especially* to government channels — the maintainer's explicit exclusion.
2. **No cellular interception.** No GSM/CDMA/LTE/5G bands in `scan-ranges.json` defaults. No IMSI catching, no SUPI capture, no paging-channel logging. The SDRs can't really do useful cell anyway; this is a posture statement, not just a capability statement.
3. **No transmit. Ever.** The rig is receive-only. The SDRs are receive-only by hardware. No transmit dongle gets bolted on. The rig may never key up on any frequency for any reason — even on bands where the maintainer holds a license (Ham, GMRS), transmit gear is operator-driven, not appliance-driven.
4. **No data exfiltration of recorded content beyond the BMN-controlled NAS.** The WG tunnel goes to BMN VPN hub → operator NAS. There is no path that uploads recordings to any third-party service, cloud, or public location. The web UI's download buttons serve recordings to the mgmt LAN only — never to the captive portal subnet, never to the public internet.
5. **No publishing.** §605 specifically prohibits divulging content of certain communications. The web UI is local-only (no public surface). Recordings remain in BMN's possession (Pi → NAS). If the maintainer ever wants to publish a recording publicly (a clip on a blog post, a YouTube demo, etc.), that's an explicit, intentional human action outside this rig's automation — and decisions there belong to the maintainer + a quick read of who/what was on the recording.
6. **No imitation of broadcast-licensed services.** The rig receives broadcast FM/AM if you ask it to, but the brief explicitly excludes broadcast FM from the scan presets, and we don't dress up the UI to look like a broadcast service.

### §13b.3 — Hard YES list (RF role)

1. **Monitor any unencrypted modulation on any band the SDR can hear.** Within scope: NBFM, WBFM, AM, SSB, packet, APRS, AIS, ADS-B, P25 Phase 1 (unencrypted), DMR (unencrypted talkgroups), NXDN (unencrypted), D-STAR (unencrypted), M17, etc. Both analog and digital.
2. **Decode and log all unencrypted control data:** CTCSS (50 standard tones, 67–254 Hz), DCS digital codes, P25 NAC, DMR color codes, talkgroup IDs, station ID burst voices, callsigns, ANI/PTT-IDs, MDC1200, Fleetsync, Quik-Call II tones, etc. — anything the radio puts on the air to manage its own operation. None of this is legally protected.
3. **Default scan presets target lawfully-monitorable bands:** Ham VHF (144–148 MHz, Part 97), Ham UHF (420–450 MHz, Part 97), CB (26.965–27.405 MHz, Part 95), MURS (151/154, Part 95), GMRS/FRS (462/467, Part 95E), marine VHF (156–162 MHz, Part 80 — RX only since BMN holds no maritime license), NOAA WX (162.400–162.550 MHz, Part 15 RX), public business VHF/UHF, airband AM RX (108–137 MHz, Part 87 — RX only). Trunked systems (P25 control channels, DMR control channels, EDACS, MotoTrbo, Connect+) tracked when present.
4. **Record audio bursts only when carrier active.** Per-transmission WAV file, named per the architect brief. No giant continuous recordings unless explicitly enabled in `config.toml`.
5. **Per-recording metadata always includes** frequency, band classification, modulation, encryption-observed flag, SDR serial that captured it, scan-pass context, geographic/event tag if the maintainer set one for the event. This metadata is the audit trail.
6. **Local-only operation by default for the RF role too.** RF web UI binds to `wlan0` mgmt only. OpenWebRX binds to `wlan0` only (proxied through nginx). No portion of the RF role is reachable from `eth0`-facing captive portal users.
7. **Off switch is fast and obvious for RF.** A `/admin/services/scanner/stop` button takes the scanner offline immediately; a `/admin/services/openwebrx/stop` button kills the live receiver. SD card has a "panic shutdown" GPIO option (Q-PANIC-GPIO) for the more paranoid the maintainer mode.
8. **Recordings respect the user's filesystem.** Default rotation policy (Q-RECORDING-RETENTION) ages out the oldest recordings when thumb drive crosses 80% full. No surprise full-disk events.

### §13b.4 — Venue / event guidelines (RF)

- The rig is receive-only; it does not affect the RF environment. Nothing to coordinate with venue RF managers.
- At venues that publish "no recording" policies aimed at attendee phones/cameras, the RF rig is a separate question — those policies generally cover *video/audio of people*, not *over-the-air radio reception*. Use judgment; if questioned, the rig is doing what any commercial scanner sold at Bass Pro Shops does.
- At venues where the maintainer is a guest of an organizer (a horse show where he's hired to set up wifi, a conference where he's speaking), the always-on RF survey is great context for "what other tech is sharing my air" — and the maintainer should mention it casually to whoever's responsible for venue RF if it comes up.
- DefCon culture is RF-curious and friendly; the rig is on-brand there. At more buttoned-up venues (corporate offices, government facilities), default to leaving the rig off and the SDRs disconnected unless there's a clear reason.

### §13b.5 — What the rig is and isn't

- It **is**: a personal/learning RF survey appliance, a tool for understanding the RF environment around BMN-managed sites, a useful "what radios are in use here" reconnaissance for radio programming jobs (the maintainer's horse-show use case), a fun toy for DefCon-style learning.
- It **is not**: a surveillance tool aimed at specific people, a wiretapping device, a tool for "listening in on" private communications in any meaningful sense (those communications aren't private — they're broadcast on public airwaves with no encryption — but framing matters), a publishing platform.

When the framing slips toward "let's listen in on what the cops are saying about us" or similar — stop and re-read §13b.5. The rig is for understanding the RF environment, not for targeting people.

---

## §14 — Open questions for the maintainer before implementation

**Answered in this v2 draft (2026-05-25):**
- ✅ Q-LICENSE: Community Use (interim placeholder text in `LICENSE`; canonical CU pending in `_licenses/CU.md` per `[any] [Global]` backlog item).
- ✅ Q-NAS: `operator NAS` ((internal IP)), `/volume1/defcon-rickroll/`.
- ✅ Q-VIDEO: the maintainer supplies legal MP4.
- ✅ Q-DISCLOSURE: Yes — Phase 8 ships `/transparency` linked from portal footer.
- ✅ **Q-LAPTOP-SCOPE (new):** Sub-folder `05_LAPTOP_PLAYBOOK/` now; promote to sibling project later.
- ✅ **Q-PROJECT-NAME (new):** Rename to "BMN RF Survey Rig (w/ DefCon Mode)" — folder rename pending (see GLOBAL_BACKLOG `[Mac]` + `[ThinkStation]` items + the physical-rename item).
- ✅ **Q-ANTENNA (new):** Option B — two dedicated antennas. Specific models pinned at parts-in-hand time.
- ✅ **Q-RF-POSTURE (new):** Receive-only on any band the SDR can hear; decode all unencrypted; no encryption breaking; no cellular; no transmit. §13b is the canonical statement.
- ✅ **Q-DESIGN-FORM (new):** Archive v1, write fresh v2 (this doc).

**Still open from v1 (apply to portal role):**

| Tag | Question | Why it matters | Status |
|---|---|---|---|
| Q-NAS-AUTH | Restricted `command="rsync ..."` account or full SSH user? | NAS-side setup runbook (Phase 11). | ❓ Default: restricted account. |
| Q-BATTERY | Battery confirmed = the maintainer's Anker. Pocket bank for DefCon? | Phase 13. | 🟡 Anker confirmed; pocket banks TBD. |
| Q-CLIENT-VOLUME | Concurrent captive portal clients per event? | Pi/Caddy worker count, DHCP pool, lease time. | ❓ Default: 100. |
| Q-EVENT-ID | Event ID format. | Filenames + NAS folder structure. | ❓ Default: `evt-YYYY-MM-DD-<slug>` for portal; `rfe-YYYY-MM-DD-<slug>` for RF. |
| Q-SOCIAL-CACHE | Local mirror scope. | Phase 9. | ❓ Default: minimal hand-built page w/ QR. |
| Q-AUTH | Admin auth — Basic + IP allowlist, or add TOTP? | Phase 4. | ❓ Default: Basic + allowlist. |
| Q-OPADDR | Operator-only interface scheme — Pi `wlan0` only? the maintainer's existing home wifi too? | Phase 1. | ❓ Default: wlan0 mgmt AP only; home wifi as alt mgmt path via Pi USB-tether if needed. |
| Q-SSIDS | Initial DefCon SSID names. | Phase 12. | ❓ Default: "Free WiFi - Lobby", "Conference WiFi", "Hotel Guest", "Coffee Shop Free WiFi". |
| Q-UPLOAD-CADENCE | Auto-upload when WG window opens, or operator-command-only? | Phase 11. | ❓ Default: auto, 10-min cooldown. |
| Q-MAP-CREDS | mAP lite BMN-onboarded yet? | Blocks Phase 2. | ❓ Default: onboard during Phase 0. |

**New open questions for v2:**

| Tag | Question | Why it matters | Status |
|---|---|---|---|
| **Q-WG-ENDPOINT** | Confirm exact AllowedIPs scope for wg0 back to BMN VPN hub. Currently sketched as `(internal IP)/16, 172.16.0.0/12` — too wide? Trim to specific BMN-internal CIDRs (per internal convention) network topology? | Phase 2 + 11. | ❓ Default: trim to `(internal IP)/16` + `(internal IP)/24` + `(internal IP)/24` (just home + Valley Mills DC). |
| **Q-SCAN-LANG** | Python (matches v1 stack) or Rust (more headroom for sustained DSP) for the scanner service? | Phase 5. | ❓ Default: Python; rebuild scanner only in Rust if Pi can't sustain hop+detect+demod+log loop at target dwell time. |
| **Q-RF-DB** | One unified SQLite (rfrig.db) for both roles, or two separate (rickroll.db + rf_events.sqlite)? | Phase 1 + 5. | ❓ Default: two separate. Failure isolation; matches the architect brief's path explicitly. |
| **Q-OPENWEBRX-INSTALL** | Install OpenWebRX as a deb package or as a Docker container? | Phase 3. | ❓ Default: Docker. Easier upgrade story; better isolation; cost is a small USB-latency penalty. |
| **Q-OPENWEBRX-AUTH** | OpenWebRX behind nginx basic auth, or rely on its own auth + IP allowlist? | Phase 3. | ❓ Default: behind nginx basic auth too; defense in depth. |
| **Q-RECORDING-RETENTION** | Rolling-by-days or rolling-by-disk-pct or keep-until-manual-purge? | Phase 6. | ❓ Default: rolling-by-disk-pct, age out oldest when /mnt/usbdata crosses 80% full. Configurable in `config.toml`. |
| **Q-LAPTOP-OS** | Win10 enterprise stays as the lab OS, or do we WSL2/dual-boot to Linux for librtlsdr-native? | Phase 0 — informs the laptop playbook contents. | ❓ Default: stay on Win10 + Zadig + native Windows RTL-SDR tools (SDR# / SDR++ / Universal Radio Hacker); WSL2 only if a specific tool we need is Linux-only. |
| **Q-PANIC-GPIO** | Wire a hard "panic stop" GPIO button that immediately halts scanner + OpenWebRX (and triggers full shutdown after long-press)? | Phase 13 polish. | ❓ Default: defer to Phase 14 polish; not required for v1 ship. |
| **Q-DEFCON-CONFIRM-UX** | DefCon toggle 2-click confirm is good; do we also add a venue picker (drop-down: "DefCon / Horse show / Office / Other") that pre-selects SSID list + duration cap? | Phase 4 polish. | ❓ Default: keep toggle simple in v1; add picker in Phase 14 if the maintainer wants. |
| **Q-LAPTOP-PROMOTE** | When does `05_LAPTOP_PLAYBOOK/` graduate to a sibling project (per the maintainer's "both — sub-folder now, promote later" decision)? Trigger conditions? | Phase 14. | ❓ Default trigger: when the playbook accumulates either (a) >10 documented experiments or (b) reusable scripts/configs that are NOT tied to this rig specifically. |

---

## Appendix A — File list this design implies (eventual repo)

```
BMN RF Survey Rig (w/ DefCon Mode)/      # folder rename pending
  MASTER_CONTEXT.md
  HANDOFF.md
  CLAUDE.md
  LICENSE                                # Community Use (placeholder until _licenses/CU.md)
  00_DESIGN.md                           # this file — v2
  00_DESIGN_v1.md                        # frozen v1 snapshot
  01_RUNBOOKS/
    pi-base-image.md                     # Phase 1
    routeros-dual-role-config.md         # Phase 2 — paste-ready RouterOS export
    openwebrx-deploy.md                  # Phase 3
    web-ui-deploy.md                     # Phase 4
    scanner-deploy.md                    # Phase 5
    tone-detector-deploy.md              # Phase 6
    portal-deploy.md                     # Phase 8 (carry-forward from v1)
    social-mirror-deploy.md              # Phase 9 (carry-forward from v1)
    nftables-hardening.md                # Phase 10
    wg-sync-setup.md                     # Phase 11 (terra-prime side too)
    nas-upload-setup.md                  # Phase 11 (operator NAS side)
  02_ROUTEROS/
    rfrig.rsc                            # full config export
    enable-defcon.rsc                    # toggle scripts
    disable-defcon.rsc
  03_PI/
    app/
      portal/                            # v1 portal blueprint
      rf/                                # new RF blueprint
    scripts/
    systemd/
    nginx/
    migrations/
    udev/
  04_HARDWARE/
    bom.md                               # BOM updated for SDRs + thumb drive + antennas
    battery-runtime-bench.md             # field test results
    antenna-options.md                   # Option B candidates + parts-in-hand decision log
  05_LAPTOP_PLAYBOOK/                    # NEW — sub-folder for Win10 laptop RF capture playbook
    README.md
    01_zadig-driver-setup.md
    02_sdr-software-survey.md
    03_first-capture-walkthrough.md
    99_PROMOTE_LATER.md                  # checklist of triggers + plan to spin into sibling project
  06_RF_DETECTORS/
    ctcss-goertzel-spec.md
    dcs-viterbi-spec.md
    digital-mode-flags.md                # which control-channel data we capture per mode
  07_BAND_PRESETS/
    scan-ranges.default.json             # seeded into /opt/bmn-rf/config/scan-ranges.json
    channels.default.json                # seeded into /opt/bmn-rf/config/channels.json
    fcc-band-table.md                    # the annotated US band table from the the architect brief
  
    NEXT_TASK.md
    LAST_RESULT.md
```

---

## Appendix B — Out of scope

Listed so future-Claude doesn't accidentally try them.

**Carry-forward from v1:**
- Real Mastodon mirror or content scraper.
- TLS cert for `social.bytemenetworks.com`.
- 5GHz captive portal operation (mAP lite's 5 GHz is reserved for STA mode → WAN uplink; the 2.4 GHz radio carries the captive portal SSIDs).
- Roaming / mesh between multiple mAP lites.
- IPv6 for captive portal clients.
- Logging of any inter-client traffic, even metadata.
- Auto-deletion of old event data on the Pi.
- Push notifications to operator on connection events.

**New for v2 (RF side):**
- Cellular interception (47 USC §605(a) — explicitly excluded).
- Encrypted-comms decryption (§13b.2 hard NO).
- Transmit on any band, ever (§13b.2 hard NO).
- Direction-finding (DF) — not in scope for v1; future enhancement if BMN buys a second SDR + antenna array for TDOA, which is a separate project.
- Spectrum-warrior pursuit modes ("follow this freq across hops") — out of scope; we record what we hear, we don't chase moving targets.
- Real-time push of recordings to BMN home — uploads are batched on the WG window per §9.2.
- Public broadcast FM (the brief excluded it; we honor that).
- IMSI catching, cell-tower simulation, anything that involves transmitting (covered by "no transmit ever").

---

*End of 00_DESIGN.md v2.*
