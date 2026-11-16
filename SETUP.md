# TrueNAS home server: how it's set up

Snapshot of the setup as of 2026-11-07. No passwords or tokens are stored in this file; the
"Secrets" section only says where they live.

## Quick reference

| What | Address |
|---|---|
| Immich (photos) | https://photos.rajatdev.fun |
| TrueNAS web UI | https://nas.rajatdev.fun |
| TrueNAS web UI, fallback if the proxy is down | https://192.168.1.12:8443 (or http://192.168.1.12:8080) |
| Nginx Proxy Manager admin | http://192.168.1.12:30020 |
| SSH | `ssh -i ~/.ssh/truenas_ed25519 truenas_admin@192.168.1.12` |

All of these only work on the home network. Nothing is exposed to the internet.

## Hardware

- Old laptop: Intel i5-8250U (4 cores / 8 threads), about 7.5 GB RAM.
- Boot drive: Geonix 256 GB SATA SSD (holds TrueNAS only).
- Data drives: two Seagate 4 TB disks, each in a LaCie d2 Professional USB enclosure.
  - ST4000NE001, serial **WS237AL2**
  - ST4000NE0025, serial **ZC13KDDM**
  - Both must be on USB 3 ports (5 Gb/s). On USB 2 the pool's write speed drops to about 38 MB/s.
  - Linux names (`sdb`, `sdc`) can swap between reboots. Use the serial numbers to tell the disks apart.
- TrueNAS version: 25.10.7.

## Network

- The laptop has no WiFi support in TrueNAS, so it's wired to a WiFi-to-Ethernet bridge.
- Interface `enp2s0` (Realtek r8169), address **192.168.1.12/24**, gateway and DNS 192.168.1.1.
- Link speed is **100 Mb/s** (the bridge's port maximum), so network copies top out around 11 MB/s.
  - If it ever shows 10 Mb/s (`cat /sys/class/net/enp2s0/speed`), reseat or replace the cable.
- The DNS records point at 192.168.1.12. If the NAS gets a different address, both domains break.
  A DHCP reservation for the NAS on the router would prevent that.

## Access and security

- Admin user: `truenas_admin`. `sudo` asks for that user's password.
- SSH service is on and starts automatically. Password login is off; only the key
  `~/.ssh/truenas_ed25519` on the Mac can log in.
- The TrueNAS UI was moved from ports 80/443 to **8080/8443** so Nginx Proxy Manager can use 80/443.

## Storage

- Pool **`tank`**: 2-way ZFS mirror of the two 4 TB disks. Either disk can fail without data loss.
- Usable space: about 3.5 TiB.
- Datasets:
  - `tank/immich/library`: Immich photos, thumbnails, encoded videos, and its daily DB backups (`backups/`).
  - `tank/ix-apps/...`: app data managed by TrueNAS (Immich database, ML cache, Nginx Proxy Manager data and certs).
  - `tank/.system`: TrueNAS system dataset.
- Scrub: checked every Sunday 00:00, runs if the last scrub is older than 35 days (about monthly).
- SMART: both disks report PASSED through the USB enclosures. ZC13KDDM once hit the
  `Spin_Retry_Count` threshold in the past (raw count 0, likely a power issue at spin-up). Keep an eye on it.
  TrueNAS 25.10 has no SMART test scheduler, so tests are run by hand
  (`sudo smartctl -t long /dev/sdX`, then `sudo smartctl -l selftest /dev/sdX`).
- Measured speeds (empty pool, both disks on USB 3): write 183 MB/s, read 214 MB/s (1 stream) to 220 MB/s (4 streams).
- Not set up yet: snapshot schedule, SMB shares, off-site backup.

## Apps

Apps run on pool `tank`.

### Immich

- Immich v3.3.6 (TrueNAS community catalog app version 1.14.62), web port 30041.
- Photos stored at `/mnt/tank/immich/library` (host path). Database and ML cache are TrueNAS-managed volumes.
- Machine learning on (CPU image). Limits: 4 CPUs, 4096 MB RAM. Timezone Asia/Kolkata.
- The first account created is the admin. There are no sign-up or invite links; the admin creates other
  users under Administration → Users.

### Nginx Proxy Manager

- v2.16.0 (catalog app version 1.3.15), on ports 80 and 443, admin UI on 30020. Limits: 2 CPUs, 1024 MB RAM.
- Runs as root (UID 0) with `SKIP_CERTBOT_OWNERSHIP=true`. Without these, first start hangs for a long time
  while it changes ownership of thousands of files on the USB disks.
- Proxy hosts (both: force HTTPS, HTTP/2, websockets on, block common exploits, HSTS off):
  - `photos.rajatdev.fun` → `http://192.168.1.12:30041` (upload limit 50000M, 600 s timeouts)
  - `nas.rajatdev.fun` → `http://192.168.1.12:8080` (no upload limit, 3600 s timeouts)
- Certificate: one Let's Encrypt ECDSA certificate for both names, issued with the Cloudflare DNS challenge
  (works without opening any ports). Expires 2027-02-05; Nginx Proxy Manager renews it automatically.

## DNS

- Domain `rajatdev.fun`, registered at Hostinger, DNS hosted on Cloudflare.
- A records `photos` and `nas` → 192.168.1.12, both **DNS only** (grey cloud). Turning on Cloudflare's
  proxy (orange cloud) would break them, because Cloudflare can't reach a home IP.
- A Cloudflare API token limited to "Edit zone DNS" on `rajatdev.fun` is stored in Nginx Proxy Manager
  for certificate renewals.

## Secrets (locations only)

| Secret | Where |
|---|---|
| SSH key for the NAS | `~/.ssh/truenas_ed25519` on the Mac |
| Nginx Proxy Manager admin login (rajatkhirid@gmail.com) | `~/.npm_admin` on the Mac (mode 600) |
| Cloudflare API token | Inside Nginx Proxy Manager; a copy is still at `~/.cloudflare_token` on the Mac (can be deleted) |
| Immich database and Redis passwords | Randomly generated, stored only in the Immich app config on the NAS |

## Known issues and workarounds

- **Slow downloads from GitHub's container registry (ghcr.io).** On this internet connection (Airtel),
  ghcr.io image downloads run at 25–55 KB/s. Immich's images were pulled through the mirror
  `ghcr.nju.edu.cn` instead, using `app.image.pull` with the same image digests, then re-tagged by pulling
  the original `ghcr.io/...@sha256:...` names (instant, since the layers were already local).
  App updates may hit the same problem; use the same workaround.
- **USB disks.** USB enclosures can briefly disconnect under load, which ZFS treats as a disk failure.
  Shut down the NAS before moving cables. After any reconnect, check `zpool status tank`.
- **Docker socket.** `truenas_admin` can't use `docker` directly. Use `midclt` calls
  (for example `app.get_instance`, `app.container_log_follow`) or the web UI for app logs.

## Planned next steps

- Import the Google Photos library: Google Takeout export (zip, 50 GB parts), then
  `immich-go upload from-google-photos --server=https://photos.rajatdev.fun --api-key=... takeout-*.zip`
  (try `--dry-run` first).
- Add a snapshot schedule for `tank/immich`.
- Set up an off-site backup.
- Optional: Tailscale or Cloudflare Tunnel for access away from home.
- Optional: set the TrueNAS system timezone (currently America/Los_Angeles) to Asia/Kolkata.
