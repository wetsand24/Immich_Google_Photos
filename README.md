# Google Photos to Immich (Google Takeout + immich-go + phone backup)

The steps and commands from my video, in the order I run them. Tested on 2026-11-09 with:
Debian 13, Docker 29.9.3, Immich v3.3.6, immich-go v0.32.0, the Immich Android app v3.3.6.

▶️ Video: https://youtu.be/OvFchxPb-2w

## What you need
- A computer that stays on, with Docker (a mini PC or an old laptop works).
  No Docker yet? Install it with Docker's official guide first: https://docs.docker.com/engine/install/
  Check: `docker --version` and `docker ps` should both work.
- At least 6 GB RAM (Immich's minimum, 8 GB recommended) and enough disk for all your photos plus a bit extra
- Your Google account

## 1. Install Immich
```bash
mkdir immich && cd immich
wget -q -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -q -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
nano .env
```
Optional: look at the compose file read-only with `nano -v docker-compose.yml` (Ctrl+W = search, Ctrl+L = move the
found line to the middle/top, Ctrl+X = close). It runs four containers: immich-server, immich-machine-learning, redis
(Valkey) and the Postgres database. Your photos (`UPLOAD_LOCATION`) and the database (`DB_DATA_LOCATION`) are folders on
the server's own disk, so they survive updates and even deleting the containers.

In `.env` (Ctrl+W to search, Ctrl+O + Enter to save, Ctrl+X to close):
```
UPLOAD_LOCATION=./library          # where your photos are stored
DB_DATA_LOCATION=./postgres
TZ=America/New_York               # your time zone (uncomment the line; New York is just the example)
IMMICH_VERSION=v3
DB_PASSWORD=ChangeMe2026           # letters and numbers only (A-Za-z0-9)
```
```bash
docker compose up -d
docker compose ps --format "table {{.Service}}\t{{.Status}}"
```
Wait until all four say `(healthy)`.

## 2. Create your account
Open `http://YOUR-SERVER-IP:2283`. The first account you create is the admin.

## 3. Export from Google (Takeout)
takeout.google.com → **Deselect all** → tick **Google Photos** → Next step → Send download link via email, Export once,
file type `.zip`, the biggest size → **Create export**. Google says it can take hours or even days. The download link
works for about a week.

Copy the zip file(s) to the server with **WinSCP** (Windows), FileZilla, or any file transfer tool, or from a terminal:
```bash
scp takeout-*.zip YOUR-USER@YOUR-SERVER-IP:~/
```
(Or run immich-go on the computer that already has the zips; there are Windows/macOS builds.)

## 4. Make an API key
Immich → profile icon → **Account Settings** → **API Keys** → **New API Key** → name it `immich-go` →
tick **Select all** → Create. Copy it now, Immich shows it only once. On plain `http://` the "Copy to Clipboard"
button can fail (browsers only allow it on https); select the text and copy it yourself.

## 5. Import with immich-go
```bash
wget -q https://github.com/simulot/immich-go/releases/download/v0.32.0/immich-go_Linux_x86_64.tar.gz
tar xzf immich-go_Linux_x86_64.tar.gz
KEY=paste-your-api-key-here

# dry run first: shows what it would do, uploads nothing
./immich-go upload from-google-photos \
  --server=http://YOUR-SERVER-IP:2283 \
  --api-key=$KEY \
  --no-ui --dry-run \
  takeout-*.zip

# the real import: same command without --dry-run
./immich-go upload from-google-photos \
  --server=http://YOUR-SERVER-IP:2283 \
  --api-key=$KEY \
  --no-ui \
  takeout-*.zip
```
- No need to unzip. If Takeout gave you several zip parts, pass them all in one run (`takeout-*.zip` does that).
- `--no-ui` prints plain text instead of the full-screen progress view.
- immich-go reads Takeout's `.json` side files, so photos keep their original date, place and albums.
- Windows / macOS builds are on the same release page: https://github.com/simulot/immich-go/releases

## 6. Phone backup
Install the Immich app (Play Store / App Store). Server address: `http://YOUR-SERVER-IP:2283`
(no `/api` at the end any more). Log in → cloud icon → pick the albums (I took Camera) → Enable Backup.
It uploads on Wi-Fi by default.

iPhone: same app and steps. Immich's docs say you must turn on **Background App Refresh** for it
(Settings → General → Background App Refresh), or it can't back up in the background:
https://docs.immich.app/features/mobile-backup

## Good to know
- Check everything is in Immich **before** you delete anything from Google. Deleting in Google Photos also deletes the
  photo on every phone that still backs up to Google.
- Immich backs up its database every night (2 AM, keeps 14), but **not your photos**. Keep another copy on another
  drive (3-2-1: three copies, one somewhere else).
- Away from home: put Immich behind a reverse proxy or a tunnel (I have videos on Nginx Proxy Manager + Tailscale and
  on Cloudflare Tunnel).

## Links
- Immich install: https://docs.immich.app/install/docker-compose
- Immich bulk upload (recommends immich-go): https://docs.immich.app/features/bulk-upload
- immich-go: https://github.com/simulot/immich-go
- Google Takeout: https://takeout.google.com
