# raspberra — homelab runbook

Everything needed to rebuild this Raspberry Pi from a blank SD card, plus how to
change or extend it afterwards. This file is the `stack` repo: the repo that
describes the machine and, through `compose.yaml`, assembles it.

**What runs here**

| Stack | Services | Reachable at |
| --- | --- | --- |
| media | gluetun (VPN), qBittorrent, Prowlarr, Plex, torrent-finder | `media.bjorngreen.se`, LAN ports |
| matte-vm | FastAPI + SQLite + PWA | its own Cloudflare hostname |
| (others) | added per the template in §10 | one hostname each |
| shared | cloudflared — **one** tunnel for the whole box | — |

**Design decisions, so the layout is not mysterious later**

- **One Compose project, assembled with `include:`.** One `docker compose up -d`
  brings up the machine, and everything shares one network, so containers reach
  each other by service name with no external network to maintain.
- **No git submodules.** Each app keeps its own repo and its own compose file,
  independently deployable; `~/stack` only points at them. Submodules would add
  a commit-the-pointer step to every app change and buy nothing on a single box.
- **One tunnel, one cloudflared.** Adding a hostname is then always the same
  action in the same place. Apps must not ship their own cloudflared.
- **App code and app data never live on the SD card.** Container config and
  media go on the SSDs; the card holds the OS and the git checkouts.

---

## 1. Hardware and OS

- Raspberry Pi (arm64), Raspberry Pi OS 64-bit (Bookworm or newer).
- Two USB SSDs:
  - `/mnt/ssd` — qBittorrent config, `media/` library, downloads-in-progress
  - `/mnt/ssd2` — Plex config/transcode, `share/torrents/{movies,tv,incomplete}`
- Wired ethernet. A static DHCP lease for the Pi makes LAN URLs stable.

Mount both at boot. Get the UUIDs with `lsblk -f`, then in `/etc/fstab`:

```fstab
UUID=xxxx-xxxx  /mnt/ssd   ext4  defaults,noatime,nofail,x-systemd.device-timeout=10  0  2
UUID=yyyy-yyyy  /mnt/ssd2  ext4  defaults,noatime,nofail,x-systemd.device-timeout=10  0  2
```

`nofail` matters: without it a disconnected USB SSD leaves the Pi stuck in
emergency mode with no network, which on a headless box means a keyboard hunt.
`noatime` cuts pointless writes.

```bash
sudo mount -a && df -h /mnt/ssd /mnt/ssd2
```

## 2. Dependencies

| Thing | Why | Install / check |
| --- | --- | --- |
| Docker Engine | everything | `curl -fsSL https://get.docker.com \| sh` |
| Docker Compose **v2.20+** | the `include:` top-level key | `docker compose version` |
| git | deploys are `git fetch` | `sudo apt install git` |
| python3 | autodeploy helpers, qbt password tool | preinstalled |
| curl, jq | health checks | `sudo apt install curl jq` |
| `/dev/net/tun` | gluetun's VPN device | present by default |

```bash
sudo usermod -aG docker "$USER"   # then log out and back in
id -u; id -g                      # expect 1000/1000 — the PUID/PGID everywhere
docker compose version            # must be >= v2.20.0
```

If Compose is older than 2.20, `include:` is silently unsupported — upgrade
Docker from Docker's own apt repo rather than Debian's `docker.io` package.

**Accounts and secrets to have in hand**

| Needed | Where it comes from |
| --- | --- |
| NordVPN **service credentials** | Nord dashboard → NordVPN → Manual setup → Service credentials. *Not* your login email/password. |
| Cloudflare account + `bjorngreen.se` on it | dash.cloudflare.com |
| Cloudflare tunnel token | Zero Trust → Networks → Tunnels → Create → Docker → the `TUNNEL_TOKEN` value |
| TorrentDay account | for the Prowlarr indexer (cookie auth) |
| Plex claim token | https://plex.tv/claim — valid 4 minutes, first run only |

## 3. Repository layout

```
~/stack/                          THIS repo — how the machine fits together
  README.md                       this file
  compose.yaml                    include: list + cloudflared
  media/compose.yaml              gluetun, qbittorrent, prowlarr, plex, finder
  bin/autodeploy                  poll git, redeploy what changed
  autodeploy.conf                 repo -> service map
  systemd/autodeploy.{service,timer}   installed into /etc (§11)
  .env                            ALL secrets, gitignored
  .env.example                    same keys, no values, committed
  .gitignore
~/git/matte-vm/                   app repo: Dockerfile + docker-compose.yml
~/git/torrent_finder/             app repo: Dockerfile + app/
~/git/<next-app>/                 same shape
```

Why apps stay outside `~/stack`: each one keeps a compose file that works on its
own (`cd ~/git/matte-vm && docker compose up`), which is what makes it testable
in isolation, while `~/stack` composes them into the real machine.

`include:` resolves each file's relative paths against **that file's own
directory**, so an app's `build: .` keeps meaning its own repo.

### The two compose files

Read them rather than a copy pasted here — [`compose.yaml`](compose.yaml) and
[`media/compose.yaml`](media/compose.yaml) are short and commented. What is
worth knowing before opening them:

- `compose.yaml` is an `include:` list plus the box's single `cloudflared`. Its
  `dns:` override does not break service discovery: on a user-defined network
  Docker keeps its own resolver at 127.0.0.11 inside the container and uses
  those addresses only as upstreams for external names.
- `media/compose.yaml` publishes **every** port of the VPN namespace —
  qBittorrent's and Prowlarr's included — on `gluetun`, because that is the only
  service in that namespace that may have a `ports:` block (§5).
- Anything with a default (`TZ`, `PUID`, `MAX_TORRENT_SIZE_GB`) is written
  `${VAR:-default}`; anything secret is `${VAR:?set in ~/stack/.env}`, so a
  missing value fails at start instead of booting a broken service.
- An `include:` whose repo is not cloned fails the whole project. Clone the app
  repos into `~/git` first (§7).

## 4. Host port registry

Keep this current — it is the cheapest way to avoid the "port is already
allocated" surprise when adding an app.

| Host port | Service | Notes |
| --- | --- | --- |
| 8080 | qBittorrent WebUI | published by **gluetun** |
| 6881 (tcp/udp) | torrent traffic | published by **gluetun** |
| 9696 | Prowlarr | published by **gluetun** |
| 8000 | matte-vm `app` | taken — do not reuse |
| 8010 | torrent-finder | container side is 8000 |
| 8090 | frontend (nginx) | container side is 80 |
| 32400 | Plex | `network_mode: host` |

Container ports never collide, only host ports. An app reachable solely through
the tunnel needs no `ports:` entry at all.

## 5. The rule that breaks things first

`qbittorrent` and `prowlarr` use `network_mode: "service:gluetun"`, so they have
**no network identity of their own**:

- `http://qbittorrent:8080` never resolves. Use **`http://gluetun:8080`**.
- `http://prowlarr:9696` never resolves. Use **`http://gluetun:9696`**.
- Neither service may have a `ports:` block — publishing a port on a container
  that borrows another's namespace fails with *"conflicting options: port
  publishing and the container type network mode"*. All their ports go on
  gluetun.
- Restarting or recreating gluetun restarts both of them, and interrupts every
  active torrent. Never let a routine app deploy touch it (see §10).

## 6. `~/stack/.env`

One file, every secret, `chmod 600`, gitignored. Compose reads `.env` from the
directory of the compose file it runs, so it must be `~/stack/.env` — app repos
do not get their own.

[`.env.example`](.env.example) is the complete key list with a comment on each
saying where the value comes from. Adding a secret means adding it to **both**
files, so a fresh clone still shows what is needed.

```bash
cp ~/stack/.env.example ~/stack/.env
chmod 600 ~/stack/.env
nano ~/stack/.env
```

Back `.env` up somewhere outside the Pi — a password manager entry is enough. It
is the one file that cannot be reconstructed from the repos.

**Which credential does what** — the question that always comes back:

| Setting | Checked by | Supplied by |
| --- | --- | --- |
| `UI_USERNAME`/`UI_PASSWORD` | torrent-finder, on every request | your browser |
| `QBITTORRENT_*` | qBittorrent's API | torrent-finder, server-side |
| `PROWLARR_API_KEY` | Prowlarr | torrent-finder, server-side |
| `NORDVPN_SERVICE_*` | NordVPN | gluetun |

## 7. First-time setup, in order

```bash
# 1. clone
mkdir -p ~/git && cd ~/git
git clone git@github.com:<you>/torrent_finder.git
git clone git@github.com:<you>/matte-vm.git
cd ~ && git clone git@github.com:<you>/stack.git       # this repo

# 2. secrets
cp ~/stack/.env.example ~/stack/.env && chmod 600 ~/stack/.env && nano ~/stack/.env

# 3. data directories, owned by the container user
sudo mkdir -p /mnt/ssd/{gluetun,qbittorrent-config,prowlarr-config,media/other} \
              /mnt/ssd2/{plex/config,plex/transcode,share/torrents/{movies,tv,incomplete}}
sudo chown -R 1000:1000 /mnt/ssd /mnt/ssd2

# 4. sanity-check the assembled config before starting anything
cd ~/stack
docker compose config -q && echo "compose OK"

# 5. VPN first, on its own, and confirm it is actually up
docker compose up -d gluetun
docker compose logs -f gluetun     # want: "Public IP address is ... (Sweden ...)"

# 6. the rest
docker compose up -d
docker compose ps
```

Then the per-service configuration in §8, and the tunnel in §9.

**Confirm the VPN really carries the torrent traffic** — the single most
important check on this box:

```bash
curl -s ifconfig.me; echo                            # the Pi's ISP address
docker compose exec qbittorrent curl -s ifconfig.me   # must be the VPN address
docker compose exec prowlarr curl -s ifconfig.me      # must match qBittorrent
```

## 8. Per-container configuration

### qBittorrent (`http://<pi>:8080`)

linuxserver's image prints a **new temporary password on every start** until a
permanent one is set, which would break the finder on each reboot.

```bash
docker compose logs qbittorrent | grep -i "temporary password"
```

1. Log in, then *Options → Web UI → Authentication*: set a real password and put
   it in `QBITTORRENT_PASSWORD`.
2. Same page: tick **Bypass authentication for clients in whitelisted IP
   subnets** → `172.16.0.0/12`. Docker bridge networks live in that range, so
   the finder authenticates regardless of password changes. The WebUI is inside
   gluetun's namespace and not internet-reachable either way.
3. *Options → Downloads*:
   - **Default Save Path**: `/auto_incomplete`
   - **Keep incomplete torrents in**: `/auto_incomplete`
4. *Categories* (right-click the sidebar → Add category):

   | Category | Save path | Host path | Plex sees |
   | --- | --- | --- | --- |
   | `movies` | `/auto_movies` | `/mnt/ssd2/share/torrents/movies` | `/movies2` |
   | `tv` | `/auto_tv` | `/mnt/ssd2/share/torrents/tv` | `/tv2` |
   | `other` | `/downloads/other` | `/mnt/ssd/media/other` | `/media/other` |

   The finder creates any that are missing. If you set them here instead, drop
   the `*_SAVE_PATH` variables and it follows whatever qBittorrent says.

Why `/auto_incomplete` and not `/downloads`: incomplete files inside a Plex
library get scanned half-written and pollute it, and the incomplete directory
sits on the same filesystem as the final folders so completion is a rename
instead of a cross-disk copy. It is also the disk the finder's free-space guard
measures — qBittorrent reports free space for the **default save path**, so
pointing that at ssd2 keeps the reserve honest.

**Lost the password entirely:** stop the container first (qBittorrent rewrites
its config on exit), then either clear the hash to get a fresh temporary
password, or write a known one:

```bash
cd ~/stack && docker compose stop qbittorrent
cp /mnt/ssd/qbittorrent-config/qBittorrent/qBittorrent.conf{,.bak}
sed -i '/^WebUI\\Password_PBKDF2=/d' /mnt/ssd/qbittorrent-config/qBittorrent/qBittorrent.conf
docker compose up -d qbittorrent
docker compose logs qbittorrent | grep -i "temporary password"
# or: python3 ~/git/torrent_finder/deploy/raspberry-pi/qbt_password.py
#     and paste the line under [Preferences]
```

### Prowlarr (`http://<pi>:9696`)

1. *Settings → General → Security*: copy the API key into `PROWLARR_API_KEY`.
2. *Indexers → Add Indexer → TorrentDay*, authenticate with your account cookie.
3. Run one search from Prowlarr's own *Search* tab before trusting it from code.
4. Recreate the finder so it picks up the key: `docker compose up -d finder`.

### torrent-finder (`http://<pi>:8010`, `media.bjorngreen.se`)

Nothing to configure in a UI; everything is environment. Confirm the routing
resolved against the real qBittorrent:

```bash
docker compose logs finder | grep 'qBittorrent category'
# movie -> qBittorrent category 'movies' at /auto_movies
# tv    -> qBittorrent category 'tv' at /auto_tv
# other -> qBittorrent category 'other' at /downloads/other
```

`/downloads/movies` in that output means the `*_SAVE_PATH` variables never
reached the container. Search behaviour: an empty search box browses the ticked
categories instead of matching a name. Deleting from the downloads drawer
deletes the files, which for a finished torrent is the copy Plex is serving.

### Plex (`http://<pi>:32400/web`)

First run only: uncomment `PLEX_CLAIM`, put a fresh token from
https://plex.tv/claim in `.env` (it expires in 4 minutes), start Plex, then
remove it again. Libraries: Movies → `/movies2` and `/media/movies`, TV →
`/tv2` and `/media/tv`, matching the mounts above.

## 9. Cloudflare

One tunnel, one cloudflared, hostnames added in the dashboard — the token-based
tunnel is **remotely managed**, so a local `config.yml` is ignored.

*Zero Trust → Networks → Tunnels → your tunnel → Public hostnames → Add*:

| Subdomain | Domain | Path | Type | URL |
| --- | --- | --- | --- | --- |
| `media` | `bjorngreen.se` | *(empty)* | HTTP | `torrent-finder:8000` |
| (matte) | `bjorngreen.se` | *(empty)* | HTTP | `app:8000` |

The URL is `<service name>:<container port>` — the container port, not the host
port, because cloudflared talks over the shared docker network. Leave **Path**
empty and give each app its own hostname: these apps serve assets from absolute
paths (`/static/app.js`), so a path prefix loads the page and 404s everything
else.

**Then put Access in front of anything that can act on your box.** *Zero Trust →
Access → Applications → Add a self-hosted application*, hostname
`media.bjorngreen.se`, policy **Allow → Emails → your address** (one-time PIN
needs no extra setup). The finder can download to your disk and delete from your
Plex library, so it gets Access *and* its own Basic auth. Two prompts in the
browser is both locks working, not a bug.

```bash
curl -sI https://media.bjorngreen.se | head -1
# 401 = the app answered and refused for lack of credentials (healthy)
# 502/530 = cloudflared could not reach the service — name or port wrong
# Cloudflare login page = Access is in front (expected once §9 is done)
```

## 10. Adding a new app

Checklist, then the skeleton:

1. Its repo has a `Dockerfile` and a `docker-compose.yml` that works standalone.
2. That compose file defines **only its own services** — no `cloudflared`, no
   `networks:` block, no host port that collides with §4 (or no `ports:` at all
   if it is tunnel-only).
3. `container_name` set, so the tunnel URL is predictable.
4. Add it to `~/stack/compose.yaml` under `include:`.
5. Add a line to `~/stack/autodeploy.conf`.
6. Add any secrets to `~/stack/.env` **and** `.env.example`, referenced with
   `${VAR:?set in ~/stack/.env}` so a missing value fails loudly at start.
7. Add the public hostname in Cloudflare (§9) plus an Access policy.
8. Update the port registry in §4 and the table at the top of this file.

```yaml
# ~/git/<new-app>/docker-compose.yml
services:
  <new-app>:
    build: .
    container_name: <new-app>
    environment:
      TZ: "${TZ:-Europe/Stockholm}"
      SOME_SECRET: "${NEWAPP_SECRET:?set in ~/stack/.env}"
    volumes:
      - /mnt/ssd/<new-app>:/data      # state on the SSD, never the SD card
    # ports:                          # only if you want LAN access
    #   - "80xx:8000"
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
    restart: unless-stopped
```

```bash
cd ~/stack
docker compose config -q                        # catches the mistake early
docker compose up -d --build <new-app>
```

## 11. Autodeployment

Polling, not webhooks: a `git fetch` every five minutes needs no inbound
endpoint, no self-hosted runner, and no secret that grants code execution here.

All four pieces are committed: [`bin/autodeploy`](bin/autodeploy), the repo →
service map in [`autodeploy.conf`](autodeploy.conf), and the units in
[`systemd/`](systemd). `autodeploy.conf` and the units hardcode
`/home/bjorngreen`; change both if the Pi's user is not `bjorngreen`.

```bash
sudo cp ~/stack/systemd/autodeploy.service ~/stack/systemd/autodeploy.timer \
        /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now autodeploy.timer
systemctl list-timers autodeploy.timer
journalctl -u autodeploy -f            # watch a deploy happen
~/stack/bin/autodeploy                 # run it by hand once to prove it works
```

Three details that matter more than they look:

- **`--no-deps <service>`** — without it a finder rebuild can recreate gluetun,
  which restarts qBittorrent and Prowlarr and interrupts every active torrent.
- **`config -q` before `up`** — the guard that makes one big project safe. A bad
  push fails the check and the running stack is left alone.
- **`git reset --hard`** — correct for a deploy checkout, destructive if you
  ever edit code directly on the Pi. If you do, switch it to `git pull
  --ff-only` and let it fail loudly instead.

Not covered by autodeploy, deliberately: `~/stack` itself. Infrastructure
changes restart real services, so pull those by hand (§12).

## 12. Routine operations

```bash
cd ~/stack

# one app, after a push (what autodeploy does)
docker compose up -d --build --no-deps finder

# infrastructure change (edited compose.yaml or .env)
git pull && docker compose config -q && docker compose up -d

# base image updates — media stack only, on purpose
docker compose pull qbittorrent prowlarr plex cloudflared
docker compose up -d qbittorrent prowlarr plex cloudflared
docker image prune -f

# gluetun / VPN change: recreates its tenants, interrupts torrents
docker compose pull gluetun && docker compose up -d gluetun

# state
docker compose ps
docker compose logs -f --tail=100 finder
df -h /mnt/ssd /mnt/ssd2 /
docker system df
```

Restart blast radius, worth internalising:

| Restarting | Also takes down |
| --- | --- |
| `gluetun` | qBittorrent, Prowlarr (namespace tenants) |
| `cloudflared` | every public hostname, ~10s |
| `qbittorrent`, `prowlarr`, `finder`, an app | only itself |

**Rollback** an app: `git -C ~/git/<app> checkout <good-sha>` then
`docker compose up -d --build --no-deps <service>`. Autodeploy will pull it
forward again on the next tick, so revert on GitHub if the rollback should
stick.

**Backups.** The irreplaceable parts are small:

```bash
tar czf ~/backup-$(date +%F).tgz \
  -C / mnt/ssd/qbittorrent-config/qBittorrent \
       mnt/ssd/prowlarr-config/config.xml \
       mnt/ssd2/plex/config/"Library/Application Support/Plex Media Server/Preferences.xml" \
  -C "$HOME" stack/.env git/matte-vm/data
```

Copy it off the Pi. Media is re-downloadable; `.env`, the Plex library database
and matte-vm's SQLite file are not.

## 13. Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `conflicting options: port publishing and the container type network mode` | a `ports:` block on a service using `network_mode: service:gluetun` | move the port to gluetun's `ports:` |
| `port is already allocated` | host port collision | pick a free host port, update §4 |
| finder: `Cannot reach qBittorrent at http://qbittorrent:8080` | namespace tenants have no DNS name | use `http://gluetun:8080` |
| finder: `qBittorrent login failed` | temporary password rotated on restart | permanent password + subnet whitelist (§8) |
| `502`/`530` from a public hostname | cloudflared cannot resolve or reach the service | service name and **container** port; is it in the same project? |
| Cloudflare hostname 404s all assets | a Path prefix was set | leave Path empty, use a subdomain |
| `docker compose exec qbittorrent curl ifconfig.me` shows the ISP IP | namespace sharing not in effect | check `network_mode`, recreate |
| that same command hangs | gluetun killswitch, usually DNS | `docker compose logs gluetun` |
| Plex shows half-finished files | downloading straight into a library | set the incomplete path (§8) |
| finder refuses a download for low space | free-space guard reading the wrong disk | default save path must be on ssd2 (§8) |
| finder container exits at start | a `${VAR:?...}` has no value | fill it in `~/stack/.env` |
| Pi stuck at boot with no network | an SSD did not mount | `nofail` in `/etc/fstab` (§1) |
| disk full, no obvious cause | old build layers | `docker image prune -f`, `docker system df` |

## 14. Security posture

- Nothing but Cloudflare is exposed: no router ports are forwarded, and the
  tunnel is outbound-only.
- Public apps get **two** locks: a Cloudflare Access policy and the app's own
  auth. The finder in particular can download to disk and delete from the Plex
  library, so keep `UI_PASSWORD` long, random and unique.
- qBittorrent's WebUI is inside the VPN namespace and reachable only on the LAN.
- Secrets live in `~/stack/.env` (0600, gitignored). They are still visible to
  `docker inspect` and `docker compose config` — that is a local-access
  exposure, not a network one.
- Rotate anything that has ever been pasted into a chat, an issue or a paste
  site: NordVPN service credentials, the tunnel token, `UI_PASSWORD`.
