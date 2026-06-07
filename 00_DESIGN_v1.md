> **Redaction note.** This file has been programmatically sanitized for public
> publication: maintainer agent names, internal workflow references, internal
> filesystem paths, internal IPs, and project-meta cross-references have been
> stripped or generalized. The technical content (RF design, Wi-Fi guardrails,
> portal mechanics, hardware layout) is intact. See the repo README for the
> public-facing project overview.

# 00_DESIGN — DefCon RickRoll Rpi+MapLite+BatteryBank

**Document type:** Engineering design (pre-implementation).
**Authored by:** Claude (Engineer role).
**For review by:** the maintainer (Owner) and the architect (PM/Architect).
**Drafted:** 2026-04-27.
**Status:** Draft 1 — no code or RouterOS configs produced. Awaiting the maintainer's answers in §14 before implementation.

This document is the engineering response to the maintainer's project brief. It produces all 14 outputs requested. Read §13 (Security/safety boundaries) before any other section if you are skimming — those guardrails dictate every decision in §1–§12.

---

## §0 — Brief from the maintainer (verbatim, 2026-04-27)

> Build a portable Wi-Fi prank/demo rig using:
> - MikroTik mAP lite as AP/router/captive portal controller
> - Raspberry Pi 4B as local web server, captive portal host, media host, logger, database server, admin dashboard, and export/upload worker
> - Battery bank for portable operation
>
> Functional behavior:
> 1. mAP lite broadcasts multiple open SSIDs, no passwords.
> 2. Connected client is redirected to a captive portal hosted on the Pi.
> 3. Captive portal plays a locally-hosted Rick Astley "Never Gonna Give You Up" video.
> 4. Page references the maintainer's personal links: https://social.bytemenetworks.com.
> 5. No internet at run-time → Pi serves a cache/local-mirror or fallback page for social.bytemenetworks.com.
> 6. Any link that would go to social.bytemenetworks.com on the prank network redirects to the Pi-hosted local version.
> 7. Connection metrics logged locally.
> 8. When the Pi later sees internet, it probes for a known NAS and uploads a nondestructive copy of the database / export.
> 9. Admin page: view metrics, export, manual upload, reset, upload history, health.
>
> Hard guardrails: not credential harvesting, not phishing, not malware, not traffic interception. Don't impersonate trusted secure networks. Don't capture passwords. Funny, portable, technically clean.

---

## §1 — Recommended technical architecture

### Topology

```
                                 (no internet during event)
                                       ╳
[Wi-Fi clients (phones, laptops)]
        │
        │  802.11b/g/n 2.4 GHz, multiple open SSIDs
        │  (single-radio mAP lite virtuals)
        ▼
┌────────────────────────────────────────────────┐
│  MikroTik mAP lite (RouterOS 7.x)              │
│  ─ Bridge: clients ↔ ether2                    │
│  ─ DHCP server: 10.66.6.0/24, gw 10.66.6.1     │
│  ─ HotSpot on bridge: redirect HTTP → Pi       │
│  ─ DNS: forwards walled-garden hosts to Pi     │
│  ─ Client isolation: ON                        │
│  ─ Firewall: drop client→client, drop          │
│    client→mAP lite admin ports (Winbox/SSH)    │
└────────────────────┬───────────────────────────┘
                     │ ether1 (LAN trunk)
                     │ static 10.66.6.2 ↔ 10.66.6.3
                     ▼
┌────────────────────────────────────────────────┐
│  Raspberry Pi 4B (Raspberry Pi OS Lite, 64-bit)│
│  eth0 = 10.66.6.3/24 (gw 10.66.6.1)            │
│  ─ Caddy (reverse proxy, ports 80/443)         │
│  ─ Gunicorn + Flask app (port 8000, internal)  │
│  ─ SQLite at /var/lib/rickroll/rickroll.db     │
│  ─ Static media at /var/lib/rickroll/media/    │
│  ─ Local "social.bytemenetworks.com" mirror    │
│  ─ Upload worker (systemd timer, opportunistic)│
│  ─ Admin UI bound to a separate path + auth    │
│  ─ usb0 / wlan0 (operator-only): out-of-band   │
│    management when at home / for NAS uploads.  │
└────────────────────────────────────────────────┘
```

### Why this split

- **mAP lite does L2/L3 and the HotSpot mechanic only.** RouterOS HotSpot is mature, handles the captive-portal-detection protocol all phones expect, and gets the "redirect anything HTTP" behavior for free. We don't reimplement it on the Pi.
- **Pi runs everything app-layer.** Flask + SQLite + Caddy is a stack the Pi can serve to ~50 simultaneous clients comfortably. Pi sees only the inputs we choose (HTTP requests, DHCP-lease hooks if we wire them, RouterOS API pulls), not raw client traffic — keeping the design honest about §13.
- **Single-radio mAP lite cannot serve clients and use Wi-Fi for an uplink at the same time.** The architecture treats the rig as offline-by-default. NAS uploads happen only when an operator-supplied path to internet appears (Ethernet plugged into mAP lite ether2, USB-tether on the Pi, or operator-mode where the Pi joins a known Wi-Fi via wlan0 with the prank radio temporarily off).
- **Two clearly separated networks on the Pi:**
  - `eth0` is the **client-facing** LAN bridged to the mAP lite. The Flask app binds here.
  - `wlan0` / `usb0` is **operator-only** — used at home for management and for the opportunistic NAS upload window. Admin UI is **only** reachable from operator-only interfaces (firewall enforced; see §13).

### Data flow at event time

1. Client picks an SSID, joins, gets DHCP from mAP lite.
2. Phone OS issues a captive-portal probe (e.g., `connectivitycheck.gstatic.com/generate_204` or `captive.apple.com`). RouterOS HotSpot intercepts, returns a redirect to `http://portal.local/` (or a literal IP).
3. Phone opens the captive portal mini-browser, which fetches the Pi's `/portal` page.
4. Pi logs the hit, records what it can about the device (MAC via RouterOS API or `arp`, DHCP hostname, user-agent), serves the Rickroll page.
5. Each interaction the user takes (play button, social link click, "exit") fires a small AJAX log event back to the Pi.
6. Connection metrics persist in SQLite.

### Data flow during opportunistic upload

1. Pi's "internet detector" daemon (systemd timer, every N minutes) probes a couple of known reachable hosts (`8.8.8.8:53`, `1.1.1.1:443`) over `wlan0`/`usb0`/`eth0` depending on which is "operator mode".
2. If reachable AND the configured NAS hostname resolves AND SSH/SFTP handshake succeeds, the daemon runs `rsync --append-verify` to push a snapshot of the SQLite DB plus rolled-up CSV exports to a dated event folder on the NAS.
3. Result is recorded in the local `uploads` ledger; success/failure surfaces in the admin UI.
4. Local DB is **never** deleted by the upload daemon. the maintainer resets metrics manually via the admin UI.

---

## §2 — Recommended Pi OS and package stack

### OS

**Raspberry Pi OS Lite, 64-bit, Bookworm** (current stable as of 2026-04-27).

- Lite (no desktop) — we never plug a screen into the rig.
- 64-bit — Pi 4B is ARMv8; better performance, future package compatibility.
- Bookworm — current stable, gets us systemd 252+, Python 3.11, fresh OpenSSL.

### Base packages

| Package | Purpose |
|---|---|
| `caddy` | Reverse proxy, HTTPS termination on operator-mode admin URL, redirect-to-self for clients. |
| `python3`, `python3-venv`, `python3-pip` | Flask app runtime. |
| `gunicorn` (via venv) | WSGI server in front of Flask. |
| `sqlite3` (CLI + library) | DB operations + ad-hoc inspection. |
| `dnsmasq` *(optional, see §7)* | Backup DNS on the Pi if we decide the mAP lite shouldn't carry DNS. Default: leave it OFF and use mAP lite DNS rewrites; turn on only if rewrites prove flaky. |
| `rsync` | Primary NAS upload tool. |
| `openssh-client` | SSH/SFTP for NAS. |
| `iw`, `iproute2`, `wpasupplicant` | Operator-mode Wi-Fi management. |
| `chrony` | Time sync (only matters when internet is available; logs use monotonic timestamps that are reconciled on reboot). |
| `ufw` *(optional)* | Simple firewall front for the iptables rules in §13. Enable if the maintainer prefers UFW idiom; otherwise raw `nftables`. |
| `fail2ban` | Just for SSH on operator interfaces. Comes from §7 of the manifest. |
| `unattended-upgrades` | Security updates. |

### Python deps (in a venv at `/opt/rickroll/venv`)

```
Flask>=3.0
gunicorn>=22.0
APScheduler>=3.10        # internet-detect + upload timer
SQLAlchemy>=2.0          # optional; raw sqlite3 is fine, SQLAlchemy gives migrations + nicer queries
Flask-Login>=0.6         # admin auth
python-routeros>=0.7     # RouterOS API to pull lease + HotSpot host data
requests>=2.31           # internet probe
itsdangerous>=2.1        # session signing (Flask brings its own; pinned for clarity)
```

### Filesystem layout

```
/opt/rickroll/
  venv/                    # Python venv
  app/                     # Flask app
    __init__.py
    portal.py              # captive portal blueprint
    admin.py               # admin UI blueprint
    api.py                 # internal log endpoints
    routeros.py            # RouterOS API helpers
    models.py              # SQLAlchemy models or raw SQL helpers
    templates/
    static/
  scripts/
    upload_worker.py
    internet_probe.py
    seed_event.py

/var/lib/rickroll/
  rickroll.db              # SQLite DB
  media/
    rickroll.mp4           # local video (the maintainer supplies — see §14 Q-VIDEO)
    poster.jpg             # video poster image
  social_mirror/           # see §8
  exports/                 # CSV/JSON dumps; uploaded copies live here

/etc/rickroll/
  config.toml              # event ID, NAS host, NAS path, admin password hash
  nas_known_hosts          # SSH known_hosts pinned to the NAS only
  nas_id_ed25519           # SSH key for NAS upload (mode 0600, owned by rickroll user)

/etc/systemd/system/
  rickroll-web.service     # Gunicorn + Flask
  rickroll-upload.service  # one-shot upload run
  rickroll-upload.timer    # opportunistic timer

/etc/caddy/
  Caddyfile
```

### Service user

- Create a non-login user `rickroll` (shell `/usr/sbin/nologin`).
- All Flask + SQLite + uploader processes run as `rickroll`.
- `rickroll` owns `/var/lib/rickroll/` and `/etc/rickroll/`.
- Admin SSH stays as the standard BMN `claude` admin user from (internal docs); the prank app never runs as root.

---

## §3 — Recommended web/app framework

**Flask + Gunicorn + Caddy.**

- **Flask** over FastAPI: this is a small synchronous app with form posts and HTML templates, not a high-throughput JSON API. Flask + Jinja2 templates is the right size. The portal page itself is mostly a static HTML+JS bundle; the server-side endpoints are tiny log-event POSTs and the admin dashboard.
- **Gunicorn** with 2–4 sync workers. Pi 4B has 4 cores; we leave headroom for Caddy, the upload worker, and SQLite.
- **Caddy** in front, doing:
  - HTTP on `:80` for clients (captive portal). No HTTPS for the portal — the captive-portal mini-browser hates self-signed certs and our portal must not pretend to be a trusted secure surface.
  - HTTPS on `:8443` for the admin UI on operator-only interfaces, with a self-signed cert (or the maintainer's local CA cert) that the operator's browser trusts after a one-time accept.
  - Redirect logic for the social.bytemenetworks.com hostname (see §8).

Reasons not to pick alternatives:
- **FastAPI/Starlette/uvicorn:** async overkill for a couple-handler app; harder template story.
- **Plain nginx + uWSGI:** nginx is fine but Caddy's site-block ergonomics + HTTPS-by-default are nicer for a small project; one config file.
- **Node/Express:** introduces a second runtime when Python already covers the upload worker, RouterOS API, and APScheduler cleanly.

### Caddyfile sketch

```caddy
:80 {
  # client-facing portal
  encode gzip
  root * /opt/rickroll/app/static
  reverse_proxy /portal* localhost:8000
  reverse_proxy /api* localhost:8000
  reverse_proxy /media* localhost:8000
  # default: send anything else to the portal landing
  handle {
    redir /portal 302
  }
}

# Operator-only admin (bound to the operator interface IP)
https://10.66.99.1:8443 {
  tls /etc/caddy/admin.crt /etc/caddy/admin.key   # self-signed; operator-trusted
  reverse_proxy /admin* localhost:8000
  reverse_proxy /admin/api* localhost:8000
}
```

Concrete bindings (operator IP, hostname) get pinned during build per §14 Q-OPADDR.

---

## §4 — Recommended SQLite schema

Designed around: nondestructive logging, easy CSV export, idempotent upserts on `(event_id, mac)`, and minimum PII surface area.

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

-- A discrete operational session. "Event" = "DefCon Saturday afternoon".
-- Multiple events can live in the same DB; admin UI scopes to "current event".
CREATE TABLE events (
    event_id        TEXT PRIMARY KEY,           -- e.g. 'evt-2026-08-09-defcon-sat-pm'
    label           TEXT NOT NULL,              -- human label
    started_at      TEXT NOT NULL,              -- ISO-8601 UTC
    ended_at        TEXT,                       -- NULL = still active
    notes           TEXT
);

-- One row per (event, mac). Upserted on every observation.
CREATE TABLE devices (
    event_id        TEXT NOT NULL REFERENCES events(event_id) ON DELETE CASCADE,
    mac             TEXT NOT NULL,              -- xx:xx:xx:xx:xx:xx, lowercase
    first_seen      TEXT NOT NULL,
    last_seen       TEXT NOT NULL,
    dhcp_hostname   TEXT,                       -- best-effort, may be NULL
    last_ip         TEXT,                       -- last DHCP-assigned IP we observed
    last_ssid       TEXT,                       -- which open SSID they were on
    vendor_guess    TEXT,                       -- OUI lookup, local table
    PRIMARY KEY (event_id, mac)
);

-- One row per HotSpot/portal landing. Same device may appear many times.
CREATE TABLE portal_hits (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    event_id        TEXT NOT NULL REFERENCES events(event_id) ON DELETE CASCADE,
    mac             TEXT,                       -- nullable: portal hit before we have MAC
    ip              TEXT,
    user_agent      TEXT,
    ssid            TEXT,
    seen_at         TEXT NOT NULL,
    portal_path     TEXT NOT NULL               -- /portal, /portal/exit, etc.
);

-- One row per "user clicked play" (or autoplay-permitted page-loaded-with-video state).
CREATE TABLE video_events (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    event_id        TEXT NOT NULL REFERENCES events(event_id) ON DELETE CASCADE,
    mac             TEXT,
    seen_at         TEXT NOT NULL,
    kind            TEXT NOT NULL CHECK (kind IN ('autoplay_attempt','play_click','ended','error')),
    detail          TEXT
);

-- Clicks that would have left the rig (social links, fallback link, QR landing).
CREATE TABLE link_clicks (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    event_id        TEXT NOT NULL REFERENCES events(event_id) ON DELETE CASCADE,
    mac             TEXT,
    seen_at         TEXT NOT NULL,
    link_kind       TEXT NOT NULL,              -- 'social_main','social_mastodon','qr','exit'
    target_host     TEXT
);

-- Upload ledger: never overwritten. One row per attempt.
CREATE TABLE uploads (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    started_at      TEXT NOT NULL,
    finished_at     TEXT,
    nas_host        TEXT NOT NULL,
    nas_path        TEXT NOT NULL,
    bytes_sent      INTEGER,
    status          TEXT NOT NULL CHECK (status IN ('success','failed','skipped_no_internet','skipped_no_nas')),
    detail          TEXT,
    db_snapshot_sha256 TEXT                     -- so we don't re-upload an unchanged DB
);

-- Health snapshots — written by a tiny systemd timer every 5 minutes.
CREATE TABLE health (
    seen_at         TEXT PRIMARY KEY,
    cpu_pct         REAL,
    load1           REAL,
    mem_used_mb     INTEGER,
    disk_free_mb    INTEGER,
    routeros_reachable INTEGER,                  -- 0/1
    internet_reachable INTEGER,                  -- 0/1
    nas_reachable    INTEGER                     -- 0/1
);

-- Index for the admin "recent connections" view.
CREATE INDEX devices_last_seen ON devices(event_id, last_seen DESC);
CREATE INDEX portal_hits_seen_at ON portal_hits(event_id, seen_at DESC);
```

### Notes

- **No payload, no cookies, no headers beyond user-agent.** §13 enforces this.
- **MAC is the device key, not "user."** OUI lookup gives a vendor guess (Apple, Samsung, etc.) — useful demo color, harmless privacy-wise.
- **Events scope everything.** Reset = end the current event and create a new one; raw rows stay intact for export.
- **WAL mode** so the upload worker can read while the web app writes.
- **No schema changes at runtime.** Migrations via Alembic or hand-written numbered scripts (`migrations/001_init.sql`, `002_add_X.sql`).

---

## §5 — Recommended MikroTik RouterOS configuration pattern

Target: RouterOS 7.x stable.

### Bridge & VLAN-free LAN

```
/interface bridge
add name=bridge-clients protocol-mode=none vlan-filtering=no comment="prank LAN"

/interface bridge port
add bridge=bridge-clients interface=ether2 comment="trunk to Pi"
# wlan virtual interfaces will be added by the wireless config below
```

### Wireless — multiple open SSIDs on one radio

The mAP lite has one 2.4GHz radio. RouterOS supports **virtual** wireless interfaces that share the master radio. We define one master and N virtuals. All have `security-profile=open` (no auth, no encryption).

```
/interface wireless security-profiles
add name=open authentication-types="" mode=none unicast-ciphers="" group-ciphers=""

/interface wireless
set [find default-name=wlan1] \
    band=2ghz-b/g/n channel-width=20mhz frequency=auto \
    mode=ap-bridge ssid="Free WiFi - Lobby" \
    security-profile=open disabled=no \
    wmm-support=enabled \
    wireless-protocol=802.11

# virtuals
add master-interface=wlan1 ssid="Conference WiFi" name=wlan1-virt1 \
    security-profile=open disabled=no
add master-interface=wlan1 ssid="Hotel Guest" name=wlan1-virt2 \
    security-profile=open disabled=no
add master-interface=wlan1 ssid="Coffee Shop Free WiFi" name=wlan1-virt3 \
    security-profile=open disabled=no
```

> **§13 reminder:** SSID names must be **clearly fake / generic** (e.g., "Free WiFi - Lobby"), not impersonations of *real* networks at the venue. the maintainer picks the venue-appropriate-but-not-deceptive list.

Bridge them all into `bridge-clients`:

```
/interface bridge port
add bridge=bridge-clients interface=wlan1
add bridge=bridge-clients interface=wlan1-virt1
add bridge=bridge-clients interface=wlan1-virt2
add bridge=bridge-clients interface=wlan1-virt3
```

### IP / DHCP / DNS

```
/ip address
add address=10.66.6.1/24 interface=bridge-clients comment="prank gw"

/ip pool
add name=prank-pool ranges=10.66.6.50-10.66.6.250

/ip dhcp-server
add interface=bridge-clients address-pool=prank-pool name=prank-dhcp lease-time=15m disabled=no

/ip dhcp-server network
add address=10.66.6.0/24 gateway=10.66.6.1 dns-server=10.66.6.1

/ip dns
set servers=10.66.6.3 allow-remote-requests=yes        # Pi handles DNS, OR
# alternative: keep local resolver, do static rewrites:
/ip dns static
add name=social.bytemenetworks.com address=10.66.6.3
add name=connectivitycheck.gstatic.com address=10.66.6.3
add name=clients3.google.com address=10.66.6.3
add name=captive.apple.com address=10.66.6.3
add name=www.apple.com address=10.66.6.3
add name=www.msftconnecttest.com address=10.66.6.3
add name=www.msftncsi.com address=10.66.6.3
```

(See §7 for the full captive-portal-detection list.)

### HotSpot

The HotSpot feature is what produces the "captive portal browser pops up automatically" experience.

```
/ip hotspot profile
add name=prank login-by=http-pap http-cookie-lifetime=1h \
    use-radius=no html-directory=hotspot dns-name=portal.local \
    smtp-server=0.0.0.0 split-user-domain=no

/ip hotspot
add name=prank interface=bridge-clients address-pool=prank-pool profile=prank \
    addresses-per-mac=2 disabled=no

/ip hotspot user
# We auto-bypass authentication so no one types anything; the redirect itself is the prank.
# Approach A: use trial mode (everyone gets in for X min after seeing portal once).
# Approach B: walled-garden the Pi unconditionally and skip "login," presenting the
#            Rickroll page as the portal HTML itself.
# Recommend Approach B — cleanest UX for a no-creds rig (see §7).
```

### Walled garden + redirect

Allow clients to reach **only** the Pi (10.66.6.3), block everything else. Redirect every HTTP request to the Pi's `/portal` URL.

```
/ip hotspot walled-garden
add dst-host=10.66.6.3 action=allow
add dst-host=portal.local action=allow
add dst-host=*.bytemenetworks.com action=allow comment="local mirror only — DNS points to Pi"

/ip hotspot walled-garden ip
add dst-address=10.66.6.3 action=accept
add dst-address=10.66.6.0/24 action=accept     # internal only
# everything else is redirected to portal by HotSpot's default behavior
```

### Firewall (client isolation, drop unwanted)

```
/ip firewall filter
# Drop client→client (isolation)
add chain=forward in-interface=bridge-clients out-interface=bridge-clients action=drop comment="client isolation"
# Drop client→mAP lite admin (Winbox 8291, SSH 22, web 80/443)
add chain=input in-interface=bridge-clients dst-port=22,80,443,8291 protocol=tcp action=drop
# Allow client→Pi
add chain=forward src-address=10.66.6.0/24 dst-address=10.66.6.3 action=accept
# Drop everything else from clients
add chain=forward in-interface=bridge-clients action=drop
```

### Logging hooks (RouterOS → Pi, see §6)

```
/system logging action
add name=pi-syslog target=remote remote=10.66.6.3 remote-port=5514 src-address=10.66.6.1
/system logging
add topics=dhcp action=pi-syslog
add topics=hotspot action=pi-syslog
```

### Out-of-band management

- Winbox/SSH access to the mAP lite restricted to ether1 (Pi LAN trunk only) **AND** to the operator's IP (10.66.6.2 = Pi). No client-network access.
- the maintainer resets via the mAP lite's reset hole if needed; default credentials never used (see §10b of manifest).

---

## §6 — Recommended way to pass connection / lease / HotSpot data from MikroTik to Pi

Three complementary channels — use all three for resilience.

### 6.1 — Syslog over UDP (cheap, lossy, real-time)

- mAP lite ships RouterOS log entries (`dhcp`, `hotspot`) to Pi UDP/5514.
- A tiny Python listener (`scripts/syslog_ingest.py`) parses lease events and inserts/updates rows in `devices` + `portal_hits`.
- Why syslog: zero polling, near-real-time, RouterOS speaks it natively.
- Why not *only* syslog: UDP is lossy; format is loose; we want a backstop.

### 6.2 — RouterOS API pull (authoritative, polled)

- Pi opens a long-lived RouterOS API connection (port 8728 or TLS 8729) and **polls every 30 s**:
  - `/ip dhcp-server lease print` → MAC, IP, hostname, status, expires.
  - `/ip hotspot host print` → which clients are currently active, since when, on which SSID virtual.
  - `/interface wireless registration-table print` → 802.11-level data (signal, last activity, association time).
- Reconcile with what syslog gave us; API wins on conflict.
- Library: `python-routeros` (or RouterOS' native binary protocol). Read-only credentials — the Pi never *writes* to RouterOS at runtime.

### 6.3 — HTTP from the captive portal page (richest, browser-side)

- The portal HTML, served by the Pi, includes a small JS bundle that POSTs:
  - On page load: `/api/log/portal_hit` with `event_id`, `client-supplied-id` (cookie), and any captive-portal `?mac=` / `?ip=` / `?username=` query params RouterOS appends to the redirect.
  - On video play / pause / end: `/api/log/video_event`.
  - On link click: `/api/log/link_click`.
- The MAC arrives via HotSpot redirect query string (RouterOS templates `$(mac)` into the redirect URL), so the Pi has authoritative MAC + IP + user-agent in one request.

### Why three channels

| Channel | Strength | Weakness |
|---|---|---|
| Syslog | Real-time, zero-load | Lossy, unstructured |
| API poll | Authoritative, structured | 30 s lag, depends on read-only API user |
| Captive-portal POST | Includes user-agent + click events | Only fires when user opens the portal — silent users don't show |

Combined, we capture: every device that ever requested DHCP (API/syslog), every device that opened the portal (HTTP), every interaction (HTTP), even when one channel hiccups.

### RouterOS read-only API user

```
/user group add name=ro-readonly policy=read,api,!write,!policy
/user add name=pi-reader group=ro-readonly password="<long-random-from-_credentials>"
```

Credential lives in `_credentials/defcon-rickroll-routeros.md` (created on first build (per internal convention)).

---

## §7 — Recommended local DNS / captive portal behavior

### Captive-portal-detection (CPD) probe URLs

Every modern OS hits a known URL at network-join time to check "is this internet, or am I behind a captive portal?" If the response is anything other than the expected 204/empty, the OS pops a captive-portal browser. We exploit this.

| OS / vendor | Probe URL |
|---|---|
| Android | `http://connectivitycheck.gstatic.com/generate_204`, `http://clients3.google.com/generate_204` |
| iOS / macOS | `http://captive.apple.com/`, `http://www.apple.com/library/test/success.html` |
| Windows | `http://www.msftconnecttest.com/connecttest.txt`, `http://www.msftncsi.com/ncsi.txt` |
| Firefox | `http://detectportal.firefox.com/canonical.html` |
| Linux NM | `http://nmcheck.gnome.org/check_network_status.txt`, `http://network-test.debian.org/nm` |

### Preferred behavior

- **DNS rewrite** all of those hostnames to the Pi (10.66.6.3). Done in `/ip dns static` on the mAP lite (see §5).
- The Pi's web server, on requests to `connectivitycheck.gstatic.com/generate_204` (and friends), returns **HTTP 302 Location: http://portal.local/portal**. This forces the captive portal browser open *immediately* — much better UX than relying on RouterOS's HotSpot to do it.
- For hosts we don't recognize, RouterOS HotSpot still does its standard "is this client logged in? no → redirect to portal" thing.

### Hostname assigned to the Pi

- `portal.local` — short, memorable, doesn't pretend to be a real domain.
- mAP lite's `/ip dns static` resolves `portal.local` → `10.66.6.3`.
- Caddy's site block for `:80` matches `portal.local` and any other Host header (catch-all).

### What the portal page does (UX)

1. **Hits the Pi at `/portal` with `?mac=AA:BB:..&ip=10.66.6.X&link-orig=...`** (RouterOS HotSpot template values).
2. Pi logs `portal_hit`, sets a session cookie keyed to `(event_id, mac)`.
3. Renders the Rickroll page:
   - Big lime ByteMe-branded headline: **"You've been Rickrolled by ByteMe Networks."**
   - Below the fold: short explainer that this is a demo rig, no data captured beyond connection metadata, link to the maintainer's social fallback page (§8) and a QR.
   - `<video autoplay muted playsinline>` loaded from `/media/rickroll.mp4` with a poster image.
   - Big obvious **"🔊 Unmute / Click to Continue"** button — required because most mobile browsers block audible autoplay; muted autoplay starts immediately, the click-to-unmute is the moment the prank lands.
   - "Done laughing? You can disconnect now." footer.
4. Optional: an **"Exit Wi-Fi"** button that calls RouterOS HotSpot's logout URL via the Pi (no user creds, just clears the HotSpot session). Wraps the joke nicely.

### What the portal page **does not** do

- No password fields, no email fields, no "fill out this form to win a prize" — that is phishing. **Never.**
- No popups that mimic OS UI.
- No pretending to be a known brand. The page leads with "ByteMe Networks demo" so anyone who reads it knows what's up.

---

## §8 — Recommended local cache / fallback for `social.bytemenetworks.com`

The brief asks for two things:

1. A local mirror so a connected client can see *something* when they tap the social link.
2. Any link/click that would normally hit `social.bytemenetworks.com` redirects to the Pi version while on the prank Wi-Fi.

Mastodon (likely what's at `social.bytemenetworks.com`) does not realistically mirror as a static site. Recommendation: don't try.

### Recommendation: hand-built static "fallback hub" page

Build a small static page at `/var/lib/rickroll/social_mirror/index.html` that:

- Visually echoes ByteMe's brand (lime accents, BMN logo, dark theme).
- Says: **"You're on the ByteMe DefCon prank rig — no internet from here. Once you're back online, find me at:"**
- Lists the maintainer's primary social handles as plain `<a href="https://...">` links — **opening these will fail at event time** (no internet) and the user reads the URL, or scans a QR.
- Includes a QR code for `https://social.bytemenetworks.com` (rendered at build time with `qrencode`) for easy phone-camera capture without typing.
- Stays under 100 KB total.

### How "links to social.bytemenetworks.com get redirected to the local version"

DNS rewrite chain:

1. mAP lite `/ip dns static` resolves `social.bytemenetworks.com` → `10.66.6.3` (the Pi).
2. Caddy on the Pi has a site block for `social.bytemenetworks.com` (and a wildcard for `*.bytemenetworks.com`) that serves `/var/lib/rickroll/social_mirror/`.
3. Result: any browser on the prank network requesting `http://social.bytemenetworks.com/...` lands on the Pi mirror automatically — no link rewriting in the portal HTML required.

Caddyfile fragment:

```caddy
http://social.bytemenetworks.com, http://*.bytemenetworks.com {
  root * /var/lib/rickroll/social_mirror
  file_server
  log
}
```

### Why not a real cache (`wget --mirror`, archivebox, etc.)

- Mastodon timelines are personalized + dynamic; a snapshot is misleading.
- Captive-portal mini-browsers don't run modern JS reliably; a full Mastodon mirror would render badly.
- A purpose-built fallback page is faster to build, easier to brand, and more on-tone for a prank rig.

### TLS note

- We serve the mirror over **HTTP only**. Browsers requesting `https://social.bytemenetworks.com` will get a TLS handshake failure (we don't have a real cert and won't fake one — see §13). They'll either fall back to HTTP (rare) or fail.
- The portal page links to the mirror via plain HTTP to avoid the failure.
- This is a deliberate tradeoff — we'd rather fail closed on TLS than do anything that looks like a MITM.

---

## §9 — Recommended NAS upload strategy

### Protocol: **rsync over SSH (with key auth)**

Why rsync-over-SSH wins for this workload:

| Option | Verdict | Reasoning |
|---|---|---|
| **rsync over SSH** | ✅ recommend | Idempotent, atomic with `--inplace --partial --append-verify`, encrypted in flight, key-auth fits BMN's existing SSH key model ((internal docs) + §10b), works with operator NAS (Synology, sshd available (per internal convention)). the maintainer's 2026-04-28 decision: **operator NAS is the upload target.** |
| **SFTP / SCP** | OK fallback | Simpler than rsync, but no resume on partial uploads, no checksum-based skip. |
| **SMB mount + cp** | ❌ avoid | Requires the Pi to keep a credentialed mount; messy on disconnects; no in-flight encryption unless SMB3 + signing; auth model differs from BMN's SSH-everywhere convention. |
| **Cloud sync (rclone)** | ❌ scope creep | We have a NAS; don't introduce a third destination. |

### Mechanics

1. **Snapshot first, upload second.**
   - The upload daemon takes a `sqlite3 .backup` snapshot of `rickroll.db` to `/var/lib/rickroll/exports/rickroll-<event_id>-<utc>.db`.
   - Optionally also writes CSV exports of `devices`, `portal_hits`, `video_events`, `link_clicks`, `uploads` next to it.
   - Upload reads from `exports/`, never from the live DB. SQLite WAL means reads while writes are happening are safe, but snapshotting first is cleaner.

2. **Idempotent upload via rsync.**
   - Target path: `<nas_root>/defcon-rickroll/<event_id>/`.
   - Command:
     ```
     rsync -av --partial --append-verify \
           -e "ssh -i /etc/rickroll/nas_id_ed25519 \
                   -o UserKnownHostsFile=/etc/rickroll/nas_known_hosts \
                   -o StrictHostKeyChecking=yes" \
           /var/lib/rickroll/exports/ \
           rickroll@<nas_host>:/volume1/defcon-rickroll/<event_id>/
     ```
   - rsync skips files unchanged-by-mtime+size; the SHA-256 in the `uploads` ledger is the second guard.

3. **Upload ledger.**
   - Every attempt writes a row to the `uploads` table (success / failed / skipped).
   - The daemon checks the most recent successful row's `db_snapshot_sha256` before re-uploading; identical snapshot → skip with status `success` and an empty bytes_sent.

4. **No deletes.**
   - The upload daemon never `rsync --delete`s. Local DB never deleted by upload. the maintainer resets metrics via the admin UI, which closes the current event and starts a new one — old rows stay until the maintainer explicitly clears them.

5. **Trigger conditions.**
   - **Internet probe** (TCP/UDP to `1.1.1.1:443`, `8.8.8.8:53`) → reachable.
   - **NAS probe** (TCP to `<nas_host>:22`) → reachable AND host-key matches `nas_known_hosts`.
   - **Cooldown** since last successful run > N minutes (default 10 — configurable).
   - All three conditions true → run.

6. **NAS target setup (the maintainer-side prerequisites).**
   - **NAS = operator NAS** ((internal IP)) per the maintainer's 2026-04-28 decision. Manifest §4 currently says operator NAS receives only the Software share from Penny; this rig adds a new dedicated share **`/volume1/defcon-rickroll/`** that does NOT need to replicate from or to Penny.
   - Create user `rickroll` on operator NAS (Synology DSM → Control Panel → User & Group → Create).
   - Grant the user read/write on `/volume1/defcon-rickroll/` only; nothing else.
   - Allow SSH key login for that user (DSM → Control Panel → Terminal & SNMP → enable SSH, ensure user shell is allowed).
   - Restrict via `~/.ssh/authorized_keys` `command="rsync --server -vlogDtprze.iLsfxC . /volume1/defcon-rickroll/"` + `no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty` (subject to Q-NAS-AUTH; default to restricted).
   - Pi-side key: generate an `ed25519` keypair owned by the `rickroll` service user; store private at `/etc/rickroll/nas_id_ed25519` (mode 0600); paste the public key into the NAS user's `authorized_keys` with the `command=` restriction.
   - Pin host key to `/etc/rickroll/nas_known_hosts` after first connect.

### What's in the upload

- `rickroll-<event_id>-<utc>.db` — full SQLite snapshot (binary).
- `devices.csv`, `portal_hits.csv`, `video_events.csv`, `link_clicks.csv`, `uploads.csv` — flat exports for grep/Excel.
- `metadata.json` — `{event_id, label, started_at, ended_at, host, version, sha256_of_db}`.

### What's *not* in the upload

- No raw network traffic, no PCAPs, no payload data — none of that is captured to begin with (§13).

---

## §10 — Admin UI layout proposal

**Reachable from operator-only interfaces only** (§13). Single-page-ish Flask app with a left nav and a few view templates.

```
┌──────────────────────────────────────────────────────────────────────┐
│  ByteMe DefCon Rickroll — Admin                  [● online] [● NAS]  │
├──────────┬───────────────────────────────────────────────────────────┤
│          │                                                           │
│  📊 Dash │  Current event: evt-2026-08-09-defcon-sat-pm              │
│  📡 Live │  Started 14:02 UTC (2 h 14 m ago)                         │
│  📜 Hist │  Connected now: 12   Unique today: 187   Portal hits: 354 │
│  ⬇ Export│  Video plays: 142   Social-link clicks: 38                │
│  ☁ NAS   │                                                           │
│  ⚙ Health│  ┌─ Recent connections (last 50) ───────────────────────┐ │
│  🔄 Reset│  │ time     mac (last 4)  ssid               vendor     │ │
│          │  │ 16:14    AA:31         Free WiFi - Lobby  Apple Inc. │ │
│          │  │ 16:13    71:8C         Conference WiFi    Samsung    │ │
│          │  │ ...                                                  │ │
│          │  └──────────────────────────────────────────────────────┘ │
│          │                                                           │
│          │  ┌─ Visits by SSID ──────┐  ┌─ Vendor breakdown ────────┐ │
│          │  │ Free WiFi - Lobby  92 │  │ Apple Inc.    52 %        │ │
│          │  │ Conference WiFi    47 │  │ Samsung       18 %        │ │
│          │  │ Hotel Guest        31 │  │ Google         9 %        │ │
│          │  └────────────────────────┘  └───────────────────────────┘ │
│          │                                                           │
└──────────┴───────────────────────────────────────────────────────────┘
```

### Pages

| Path | Purpose |
|---|---|
| `/admin` | Dashboard (above). Live counts, recent connections, vendor + SSID breakdowns. |
| `/admin/live` | Auto-refreshing live tail of new portal hits. |
| `/admin/events` | List of events, click into one for that event's stats. |
| `/admin/devices` | Searchable/filterable table of all devices in current event. |
| `/admin/export` | Download CSV of any table; download a `.db` snapshot. |
| `/admin/nas` | NAS upload status, history, "Upload now" button. |
| `/admin/health` | Pi health (CPU, mem, disk, temp), RouterOS reachability, internet status. |
| `/admin/reset` | "End current event + start new event" (NOT "delete data"). Two-step confirm. |

### Auth

- HTTP Basic over HTTPS (Caddy-provided cert), credentials in `/etc/rickroll/config.toml` as a `bcrypt` hash.
- *AND* IP allowlist: bind `/admin` only on the operator interface IP. (Defense in depth — if someone bridges onto operator-side, they still need the password.)
- Sessions via Flask-Login with strict cookies (`Secure`, `HttpOnly`, `SameSite=Strict`).
- See §14 Q-AUTH for whether the maintainer wants to add a TOTP factor.

### Visual style

- Lime accent on dark background, mascot in corner — pull from `BMN Auction Platform/_brand/` once that style is locked. Until then, plain CSS, no external font CDN (we're offline — everything self-hosted).
- Mobile-friendly so the maintainer can drive it from his phone after plugging in a USB-tether.

---

## §11 — Battery / runtime assumptions

### Power draw

| Device | Idle (W) | Typical (W) | Peak (W) |
|---|---|---|---|
| Raspberry Pi 4B (no peripherals) | ~3.0 | ~5.0 | ~7.5 |
| MikroTik mAP lite | ~1.5 | ~2.0 | ~3.0 |
| **Combined** | **~4.5** | **~7.0** | **~10.5** |

(Pi numbers from official Raspberry Pi Foundation guidance; mAP lite from MikroTik datasheet ~2.5 W max.)

### Battery sizing

Battery banks are typically labeled in mAh at 3.7V nominal cell voltage but deliver at 5V via a step-up. Real usable energy at the 5V output is ~70–80% of label.

| Bank label | Nominal Wh | Realistic Wh @ 5V out | Runtime @ 7 W typical |
|---|---|---|---|
| 10,000 mAh | 37 Wh | ~28 Wh | ~4 h |
| 20,000 mAh | 74 Wh | ~56 Wh | ~8 h |
| 26,800 mAh | 99 Wh | ~75 Wh | ~10 h |
| 40,000 mAh | 148 Wh | ~110 Wh | ~15 h |

### Recommendation

- **Minimum: 20,000 mAh USB-PD bank** with two outputs and ≥ 3 A on at least one port (for Pi 4B's 5V/3A requirement; the mAP lite is happy on 5V/1A).
- **Sweet spot: 26,800 mAh PD bank** — gets a comfortable full-day event with margin.
- **Cabling:**
  - Pi 4B ← USB-C, 5V/3A.
  - mAP lite ← microUSB (or PoE). MicroUSB is simpler with a battery bank; PoE requires an injector and adds parts.
- **Pass-through charging:** preferred so a wall plug "tops up" between event days without unplugging the rig.
- **TFA charge:** if the bank is FAA-checked-bag-friendly, that's a nice-to-have for travel to events.

### Operational tips

- A small **USB power monitor inline** lets the maintainer see watts pulled and remaining bank capacity in real time.
- The Pi is touchy about under-voltage; use a known-good USB-C cable (short, thick — under-voltage is usually a thin cable, not a weak bank).
- `/etc/rickroll/config.toml` flag `low_power_mode = true` can: drop Caddy access logs, throttle the API poll to 60s, drop health writes to once per 15 min. Trims a few hundred mW.

### Field check

- Validate the chosen bank against a 4-hour bench run before an event: live the rig under simulated load, log power draw, verify no thermal throttling on the Pi.

---

## §12 — Phased build plan

Each phase ends in a checkpoint the maintainer can poke at. Phases are scoped to fit in a single (task queue) per the (internal docs) protocol.

### Phase 0 — Prereqs & decisions (no code)

- License chosen (§14 Q-LICENSE).
- NAS hostname / port / user / path / key path decided (§14 Q-NAS).
- Battery bank purchased / borrowed (§11).
- mAP lite confirmed running RouterOS 7.x stable, factory-reset, default credentials cleared.
- Local copy of "Never Gonna Give You Up" sourced legally (§14 Q-VIDEO).

### Phase 1 — Pi base image

- Raspberry Pi OS Lite 64-bit installed.
- BMN standard SSH banner ((internal docs)).
- BMN authorized_keys sync configured ((internal docs)).
- `rickroll` service user, `/opt/rickroll/`, `/var/lib/rickroll/`, `/etc/rickroll/` directories created.
- Caddy + Python venv installed.
- Hostname `bmn-rickroll-pi`.

**Checkpoint:** `ssh claude@bmn-rickroll-pi` works, Caddy serves a "hello, ByteMe" page on port 80.

### Phase 2 — RouterOS base config

- mAP lite reset.
- Bridge + DHCP + DNS configured (§5 main blocks).
- One open SSID up (skip virtuals for now).
- HotSpot active, redirecting to a temporary "phase 2 placeholder" page on the Pi.

**Checkpoint:** A phone joins the open SSID, sees the captive portal pop up, lands on the Pi placeholder.

### Phase 3 — Captive portal + Rickroll video

- Flask app skeleton with `/portal`, `/portal/exit`, `/api/log/*`.
- SQLite schema deployed (§4).
- Local Rickroll video file in place at `/var/lib/rickroll/media/rickroll.mp4`.
- Portal HTML/CSS/JS shipped, including muted autoplay → click-to-unmute UX.
- RouterOS API read-only user created; API poller service running.

**Checkpoint:** Phone joins, gets Rickrolled with audio after one tap, hit appears in admin UI's recent-connections feed.

### Phase 4 — `social.bytemenetworks.com` mirror

- Static fallback page authored, branded, with QR.
- DNS rewrite for `*.bytemenetworks.com` on mAP lite.
- Caddy site block for the host.
- "Visit the maintainer's social" link on portal page works (lands on mirror).

**Checkpoint:** Tap link from portal → land on mirror; QR scan resolves to the real URL when off the prank network.

### Phase 5 — Admin UI MVP

- `/admin` dashboard with counts, recent connections.
- `/admin/export` (CSV + .db).
- `/admin/reset` (event end + new).
- `/admin/health` minimal.
- Auth + IP-allowlist + HTTPS via Caddy on operator interface.

**Checkpoint:** the maintainer can drive the admin UI end-to-end from a laptop on operator-mode.

### Phase 6 — Internet detection + NAS upload

- Internet probe service.
- NAS upload daemon (rsync-over-SSH, ledger, snapshot-first).
- Admin UI: NAS page, "Upload now" button, history.

**Checkpoint:** Plug Pi into a known internet path, watch upload trigger automatically; "upload now" button works on demand.

### Phase 7 — Multi-SSID + isolation hardening

- 3–4 virtual wireless interfaces with non-deceptive open SSID names.
- Client isolation on, mAP lite admin firewalled off from clients (§5 firewall block).
- Vendor OUI lookup table populated.

**Checkpoint:** All SSIDs land on the same portal flow; client A can't ping client B; mAP lite Winbox/SSH unreachable from client network.

### Phase 8 — Battery + portability + field test

- Tested-on-bench under battery for ≥ 4 hours.
- Carry case / strap / 3D-printed mount as the maintainer prefers.
- Final SSID list approved by the maintainer for the first event.
- "Off switch" procedure documented (RouterOS `/system shutdown`, Pi `sudo halt`).
- Dry run at home / office.

**Checkpoint:** the maintainer walks the rig out the door, confirms it works, comes back, checks the upload landed on Penny.

### Phase 9 — Polish (post first event)

- Dashboard improvements based on what was actually useful.
- Charts / export improvements.
- Logging cleanup.
- Documentation pass for repo / runbook.

---

## §13 — Security / safety boundaries

These are non-negotiable. Every other section in this document defers to this one.

### Hard NO list

1. **No credential capture.** Zero password fields, zero email fields, zero "fill out this form" anything. The portal collects nothing the user types.
2. **No phishing / no impersonation of trusted brands.** SSID names are clearly generic ("Free WiFi - Lobby"), not "Marriott Guest" or "Defcon Conference Wi-Fi" or any real venue's actual SSID.
3. **No traffic interception.** No `tcpdump`, no PCAP capture, no DNS resolver that logs queries (DNS rewrites are explicit static entries; no wildcard logging). RouterOS HotSpot's optional traffic accounting features are **off**.
4. **No TLS MITM.** No fake certs for any domain we don't own. The `social.bytemenetworks.com` mirror is HTTP only; clients hitting it via HTTPS get a clean handshake failure.
5. **No malware delivery.** The portal page is plain HTML/CSS/JS + the Rickroll video. No drive-by download, no auto-redirect to anything that could load arbitrary code.
6. **No payment / sensitive form.** "Free Wi-Fi, click for terms" is the entire UX.
7. **No content-of-traffic logging.** We log only metadata: MAC, DHCP hostname, IP, user-agent, timestamps, SSID joined, page hits, click events.
8. **No persistent tracking across events.** Reset closes the event; new event starts fresh. No cross-event MAC linking surfaced in UI (the data is there for export, but the UI scopes to current event only).

### Hard YES list

1. **Clear "this is a prank" disclosure on the portal page.** "You've been Rickrolled by ByteMe Networks. This is a demo rig. We log connection metadata only — see [link to a /transparency page on the Pi]."
2. **Client isolation on.** Clients can't reach each other or the mAP lite admin surface.
3. **Admin reachable only from operator interfaces.** Firewall enforces it on both the Pi and the mAP lite. Even with the admin password, you can't reach it from a client SSID.
4. **All admin auth over HTTPS.** Self-signed cert that the maintainer's operator devices have explicitly trusted.
5. **All keys live in OneDrive `_credentials/` (per internal convention).** No keys in code, no keys in repo, no keys in screenshots.
6. **Off switch is obvious and fast.** Two minutes from "uh, kill it" to "rig is dark."
7. **Local-only operation by default.** If something needs internet, it's an opt-in operator action.

### Venue / event guidelines

- Don't run at venues with critical real Wi-Fi (hospitals, in-flight, conferences with badging-over-Wi-Fi).
- Don't run inside a venue's controlled-access space without permission. The mAP lite is low power, but a dense crowd of phones associating to your open SSIDs creates real RF noise.
- Have ID + a one-page "what this is" handout on hand at events that ask. Includes the maintainer's contact info and the BMN URL.
- DefCon culture is friendly to clearly-disclosed prank rigs; other events may not be. When in doubt, leave it off.

### Compliance

- FCC Part 15: 2.4GHz operation under unlicensed limits, mAP lite is FCC-certified out of the box. We don't modify TX power above the device's certified levels.
- CFAA / similar laws: this rig does not access any computer it isn't asked to talk to. Clients voluntarily join an open SSID; we redirect their *own* HTTP requests to *our* server. No unauthorized access anywhere.

---

## §14 — Open questions for the maintainer before implementation

**Answered in the bootstrap session (2026-04-28):** Q-LICENSE, Q-NAS, Q-VIDEO, Q-DISCLOSURE.
**Still open:** the rest.

Answer remaining items in chat or by editing this section directly. **None of Phase 1+ should start until the still-open items affecting Phase N are resolved** — Phase 6 in particular needs Q-NAS-AUTH and the concrete `<nas_host>` value before the upload daemon can be wired.

| Tag | Question | Why it matters | Status / Decision |
|---|---|---|---|
| **Q-LICENSE** | Which BMN license category applies ((internal docs))? | Required before any code commits (per internal convention) + §14. | ✅ **Answered 2026-04-28: Community Use (CU).** Canonical CU text not yet drafted in `_licenses/CU.md`; project ships under placeholder Community Use terms in `LICENSE` until the canonical text is published. Drafting CU is now a global backlog item. |
| **Q-NAS** | NAS target: hostname/IP, SSH port, username, destination folder, key path. | Phase 6 needs all five values. | 🟡 **Partially answered 2026-04-28: NAS = `operator NAS` (secondary NAS, (internal IP) (per internal convention)).** Still needed: SSH user on operator NAS (recommend new `rickroll` user), destination folder (recommend `/volume1/defcon-rickroll/`), key path on the Pi. **Manifest §4 currently says operator NAS only receives the Software share from Penny — we'll need to enable a separate share for prank-rig exports as part of Phase 0 NAS prep.** |
| **Q-NAS-AUTH** | Restricted `command="rsync ..."` account or full SSH user? | NAS-side setup runbook. | ❓ **Open.** Default if unanswered: restricted account (recommended). |
| **Q-BATTERY** | Battery bank size — 20,000 / 26,800 / 40,000 mAh? Existing bank to plan around? | Phase 8. | ❓ **Open.** Default: 20,000 mAh USB-PD, 2 outputs. |
| **Q-CLIENT-VOLUME** | Concurrent clients per event — 10, 100, 500? | Sizing Pi/Caddy worker count, DHCP pool, lease time. | ❓ **Open.** Default: 100 concurrent, /24 pool, 15-min lease. |
| **Q-EVENT-ID** | Event ID format — `evt-YYYY-MM-DD-<slug>` or other? | Filenames + NAS folder structure. | ❓ **Open.** Default: `evt-YYYY-MM-DD-<slug>`. |
| **Q-SOCIAL-CACHE** | Local mirror scope — minimal hand-built fallback page or multi-page Mastodon-look-alike? | Phase 4. | ❓ **Open.** Default: minimal hand-built page with QR. |
| **Q-AUTH** | Admin auth: HTTP Basic + IP allowlist, or add TOTP? | Phase 5. | ❓ **Open.** Default: Basic + IP allowlist. |
| **Q-OPADDR** | Operator-only interface scheme — Pi `wlan0` to home Wi-Fi, USB-tether, or both? | How `/admin` is reached. | ❓ **Open.** Default: both. |
| **Q-VIDEO** | Source for the local Rick Astley video file. | Phase 3. | ✅ **Answered 2026-04-28: the maintainer will source a legal MP4 himself.** Build will include a transfer runbook (`scp` from operator host to `/var/lib/rickroll/media/rickroll.mp4`); no auto-download from any third-party service. |
| **Q-DISCLOSURE** | `/transparency` page on the Pi explaining what's logged + link from portal footer? | §13 hard YES item. | ✅ **Answered 2026-04-28: Yes.** Phase 3 ships a `/transparency` page; portal footer links to it; page lists exactly what each table in §4 stores. |
| **Q-SSIDS** | Initial SSID names. | Phase 7. | ❓ **Open.** Default: "Free WiFi - Lobby", "Conference WiFi", "Hotel Guest", "Coffee Shop Free WiFi". |
| **Q-UPLOAD-CADENCE** | Auto-upload when internet appears, or operator-command-only? | Phase 6. | ❓ **Open.** Default: auto, 10-min cooldown. |
| **Q-MAP-CREDS** | Has the mAP lite been BMN-onboarded ((internal docs)) yet? | Blocks Phase 2. | ❓ **Open.** Default: onboard during Phase 0. |

---

## Appendix A — File list this design implies (for the eventual repo)

```
DefCon RickRoll Rpi+MapLite+BatteryBank/
  MASTER_CONTEXT.md
  HANDOFF.md
  CLAUDE.md
  LICENSE                                # Q-LICENSE
  00_DESIGN.md                           # this file
  01_RUNBOOKS/
    pi-base-image.md                     # Phase 1
    routeros-base-config.md              # Phase 2 — paste-ready RouterOS export
    portal-deploy.md                     # Phase 3
    social-mirror-deploy.md              # Phase 4
    admin-deploy.md                      # Phase 5
    nas-upload-setup.md                  # Phase 6
  02_ROUTEROS/
    rickroll.rsc                         # full config export, importable
  03_PI/
    app/                                 # Flask app source
    scripts/
    systemd/
    caddy/
    migrations/
  04_HARDWARE/
    bom.md                               # bill of materials
    battery-runtime-bench.md             # field test results
  
    NEXT_TASK.md
    LAST_RESULT.md
```

---

## Appendix B — Out of scope

Listed here so the next session doesn't accidentally try to do them.

- Real Mastodon mirror or content scraper.
- TLS cert for `social.bytemenetworks.com` (we don't own DNS in the prank network's view of the world for any purpose other than redirect; never present a cert for that domain).
- 5GHz operation (mAP lite is 2.4GHz only).
- Roaming / mesh between multiple mAP lites.
- IPv6 for clients (RouterOS supports it; not worth it for this rig).
- Logging of any inter-client traffic, even metadata.
- Auto-deletion of old event data on the Pi.
- Push notifications to operator on connection events.
