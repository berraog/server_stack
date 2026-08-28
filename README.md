# homelab runbook

Everything needed to rebuild this server from a blank disk, plus how to change
or extend it afterwards. The hardware is a BOSGAME mini PC (Intel N95) running
Ubuntu Server 24.04. The Raspberry Pi it replaces is still a supported target
on the `pi` branch; §15 covers restructuring the disks and moving between them. This file is the `stack` repo: the repo that
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

## 1. Hardware, OS and disks

- BOSGAME mini PC, Intel N95 (Alder Lake-N, x86_64), Ubuntu Server 24.04 LTS.
  The Raspberry Pi is still a supported target — see the `pi` branch and §15.
- Wired ethernet. A static DHCP lease makes LAN URLs stable.

Three disks, three jobs:

```
/srv/appdata/          internal 500 GB SSD — all container state
  gluetun/  qbittorrent/  prowlarr/  plex/{config,transcode}/  matte-vm/

/mnt/active/           USB SSD — everything torrents touch
  incomplete/          qBittorrent's default save path AND its incomplete path
  movies/  tv/         finished, and what Plex reads
  other/               the `other` category; no Plex library

/mnt/archive/          USB SSD — content moved off active once it fills
  movies/  tv/         read-only to every container
```

**Why state is internal and media is not.** Plex's library is a SQLite database
bound by random writes, so its location is the one you can actually feel; media
is sequential, re-downloadable, and far too big for 500 GB. All three disks are
SSDs, so the mountpoints name each disk's *job* rather than its bus.

**Why one disk holds everything torrents touch.** qBittorrent gets a single
mount — `/mnt/active` as `/data` — so `incomplete/`, `movies/`, `tv/` and
`other/` are unavoidably on one filesystem. Completing a torrent is then a
rename instead of a copy, and it cannot regress into a copy later by someone
pointing one category at the other disk. The finder's free-space guard reads
qBittorrent's *default* save path, which is on this disk too, so
`MIN_FREE_SPACE_GB` guards the disk that actually fills.

That last point is why downloads stay on a USB drive rather than following the
rest of the state onto the fast internal SSD: put `incomplete/` there and every
completion becomes a cross-disk copy, while `MIN_FREE_SPACE_GB` starts guarding
500 GB of system disk as the media drive silently fills.

**Active and archive are roles, not brands.** Assign `active` to whichever drive
has more free space. When it fills, move a finished folder across by hand:

```bash
mv /mnt/active/movies/"Some Film (2019)" /mnt/archive/movies/
```

Plex finds it again because both directories are in the same library. qBittorrent
does not, so that torrent stops seeding — which is the intended trade and the
reason it is a deliberate manual step rather than a script. Two disks is also the
point at which a union filesystem (mergerfs) starts paying for itself; it is not
worth the mount-order failure modes yet.

### The installer's LVM default will hide 400 GB

Ubuntu Server's guided LVM layout gives the root logical volume ~100 GB and
leaves the rest of the volume group unallocated. Set the size during install, or
afterwards:

```bash
sudo vgs && sudo lvs                       # VFree > 0 means this bit is needed
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
df -h /
```

### Mount the USB drives at boot

Get the UUIDs with `lsblk -f`, then in `/etc/fstab`:

```fstab
UUID=xxxx-xxxx  /mnt/active   ext4  defaults,noatime,nofail,x-systemd.device-timeout=10  0  2
UUID=yyyy-yyyy  /mnt/archive  ext4  defaults,noatime,nofail,x-systemd.device-timeout=10  0  2
```

`nofail` matters: without it a disconnected USB SSD leaves the box stuck in
emergency mode with no network, which on a headless machine means a keyboard
hunt. `noatime` cuts pointless writes.

```bash
sudo mkdir -p /mnt/active /mnt/archive
sudo mount -a && df -h / /mnt/active /mnt/archive
```

## 2. Dependencies

| Thing | Why | Install / check |
| --- | --- | --- |
| Docker Engine | everything | Docker's apt repo — see below |
| Docker Compose **v2.20+** | the `include:` top-level key | `docker compose version` |
| git | deploys are `git fetch` | preinstalled |
| python3 | autodeploy helpers, qbt password tool | preinstalled (3.12) |
| curl | health checks | preinstalled |
| jq | health checks | `sudo apt install jq` |
| `/dev/net/tun` | gluetun's VPN device | present by default |
| `/dev/dri/renderD128` | Plex Quick Sync (§8) | present once `i915` loads |

**Install Docker from Docker's own repo.** Two Ubuntu-flavoured ways to get this
wrong:

- `snap install docker` — **never.** Snap's strict confinement blocks bind
  mounts outside `/home` and `$SNAP_DATA`, which is every volume in this stack,
  and the failure looks like an empty container directory rather than an error.
  If it is already there: `sudo snap remove --purge docker`.
- `apt install docker.io` — Ubuntu's package is too old. Below Compose v2.20
  `include:` is silently unsupported and the whole assembly quietly does nothing.

```bash
curl -fsSL https://get.docker.com | sh   # adds Docker's apt repo, installs both
sudo usermod -aG docker "$USER"          # then log out and back in
id -u; id -g                             # expect 1000/1000 — the PUID/PGID everywhere
docker compose version                   # must be >= v2.20.0
```

**The render group, for Plex hardware transcoding.** The gid differs between
installs, so it is a variable rather than a literal in the compose file:

```bash
ls -l /dev/dri/renderD128                # must exist
getent group render | cut -d: -f3        # e.g. 993 -> RENDER_GID in ~/stack/.env
```

**Three Ubuntu defaults worth knowing before they surprise you**

- **`unattended-upgrades` is on.** It will update `docker-ce` and restart the
  daemon, which restarts every container. `restart: unless-stopped` recovers,
  but a gluetun restart interrupts every active torrent (§12). To stop that:
  `sudo apt-mark hold docker-ce docker-ce-cli containerd.io`, and update Docker
  deliberately instead.
- **`ufw` does not filter published container ports.** Docker inserts its own
  rules ahead of ufw's chains, so anything in a `ports:` block is reachable on
  the LAN whatever ufw claims. Close a port by deleting the `ports:` entry, not
  with a firewall rule.
- **systemd-resolved** puts `127.0.0.53` in `/etc/resolv.conf`. Docker sees a
  loopback-only resolver and gives bridge containers public DNS instead, logging
  a warning at daemon start. Harmless here — it makes cloudflared's `dns:` block
  redundant rather than wrong.

**Accounts and secrets to have in hand**

| Needed | Where it comes from |
| --- | --- |
| NordVPN **service credentials** | Nord dashboard → NordVPN → Manual setup → Service credentials. *Not* your login email/password. |
| Cloudflare account + `bjorngreen.se` on it | dash.cloudflare.com |
| Cloudflare tunnel token | Zero Trust → Networks → Tunnels → Create → Docker → the `TUNNEL_TOKEN` value |
| TorrentDay account | for the Prowlarr indexer (cookie auth) |
| Plex claim token | https://plex.tv/claim — valid 4 minutes, first run only |
| Plex Pass | only for hardware transcoding; without it Plex transcodes in software |

## 3. Repository layout

```
~/stack/                          THIS repo — how the machine fits together
  README.md                       this file
  compose.yaml                    include: list + cloudflared
  media/compose.yaml              gluetun, qbittorrent, prowlarr, plex, finder
  apps/matte-vm.yaml              one fragment per standalone app
  hw/x86.yaml, hw/pi.yaml         per-hardware overlays, picked in .env
  bin/autodeploy                  poll git, redeploy what changed
  autodeploy.conf                 repo -> service map
  systemd/autodeploy.{service,timer}   installed into /etc (§11)
  .env                            ALL secrets, gitignored
  .env.example                    same keys, no values, committed
  .gitignore
~/git/matte-vm/                   app repo: Dockerfile + its own compose file
~/git/torrent_finder/             app repo: Dockerfile + app/
~/git/<next-app>/                 same shape
/srv/appdata/<service>/           container state, internal SSD (§1)
/mnt/active/, /mnt/archive/       bulk media, USB drives (§1)
```

Why apps stay outside `~/stack`: each one keeps a compose file that works on its
own (`cd ~/git/matte-vm && docker compose up`), which is what makes it testable
in isolation, while `~/stack` composes them into the real machine.

**The stack does not read those app compose files.** Each app gets a fragment
here that only `build:`s from `~/git/<app>`. That is the pattern `media/` already
used for the finder, generalised — because an included app file must not define
`cloudflared`, must not set `networks:`, and must not collide on a host port,
and an app repo that predates those rules breaks the whole assembly:

```
services.cloudflared conflicts with imported resource
```

`config -q` fails, so nothing starts and autodeploy refuses every deploy. Owning
the fragments here removes the rule instead of documenting it, and leaves the app
repos free to ship whatever compose file is convenient for standalone use.

`include:` resolves each file's relative paths against **that file's own
directory**, so `apps/matte-vm.yaml` reaching `../../git/matte-vm` works from
wherever compose is invoked.

### The two compose files

Read them rather than a copy pasted here — [`compose.yaml`](compose.yaml) and
[`media/compose.yaml`](media/compose.yaml) are short and commented. What is
worth knowing before opening them:

- `apps/matte-vm.yaml` is the template for a new app — one service, state under
  `/srv/appdata`, a healthcheck, nothing about tunnels or networks.
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

### Hardware overlays

Two things genuinely differ between the mini PC and the Pi — Plex's Quick Sync
device and the VPN cipher — so they live in [`hw/`](hw) rather than in the files
above, and `.env` picks one:

```dotenv
COMPOSE_FILE=compose.yaml:hw/x86.yaml    # or hw/pi.yaml
```

Compose reads `COMPOSE_FILE` from `.env`, so this is the only line that differs
between the two machines and every other file stays byte-identical. Two
consequences worth knowing:

- **Never pass `-f`** to `docker compose` in `~/stack`. An explicit `-f`
  overrides `COMPOSE_FILE` and silently drops the overlay — Plex would come up
  without `/dev/dri` and transcode in software with no error anywhere. Run bare
  `docker compose` from `~/stack`; `bin/autodeploy` does the same, which is why
  it `cd`s instead of using `-f`.
- Verify the overlay actually landed:
  ```bash
  cd ~/stack && docker compose config | grep -A2 'group_add\|/dev/dri'
  ```

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

Back `.env` up somewhere off the box — a password manager entry is enough. It
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
#    internal SSD: container state.  USB drives: bulk media only.
sudo mkdir -p /srv/appdata/{gluetun,qbittorrent,prowlarr,plex/config,plex/transcode,matte-vm}
sudo mkdir -p /mnt/active/{incomplete,movies,tv,other} \
              /mnt/archive/{movies,tv}
sudo chown -R 1000:1000 /srv/appdata /mnt/active /mnt/archive

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
curl -s ifconfig.me; echo                            # the box's ISP address
docker compose exec qbittorrent curl -s ifconfig.me   # must be the VPN address
docker compose exec prowlarr curl -s ifconfig.me      # must match qBittorrent
```

## 8. Per-container configuration

### qBittorrent (`http://<host>:8080`)

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
   - **Default Save Path**: `/data/incomplete`
   - **Keep incomplete torrents in**: `/data/incomplete`
4. *Categories* (right-click the sidebar → Add category):

   | Category | Save path | Host path | Plex sees |
   | --- | --- | --- | --- |
   | `movies` | `/data/movies` | `/mnt/active/movies` | `/active/movies` |
   | `tv` | `/data/tv` | `/mnt/active/tv` | `/active/tv` |
   | `other` | `/data/other` | `/mnt/active/other` | *(no library)* |

   The finder creates any that are missing. If you set them here instead, drop
   the `*_SAVE_PATH` variables and it follows whatever qBittorrent says.

Every path above is under the single `/data` mount, which is the whole active
disk (§1). That is what makes completion a rename rather than a copy, and it is
why `incomplete/` is a sibling of the library folders instead of living inside
one: incomplete files inside a Plex library get scanned half-written and pollute
it.

**Lost the password entirely:** stop the container first (qBittorrent rewrites
its config on exit), then either clear the hash to get a fresh temporary
password, or write a known one:

```bash
cd ~/stack && docker compose stop qbittorrent
cp /srv/appdata/qbittorrent/qBittorrent/qBittorrent.conf{,.bak}
sed -i '/^WebUI\\Password_PBKDF2=/d' /srv/appdata/qbittorrent/qBittorrent/qBittorrent.conf
docker compose up -d qbittorrent
docker compose logs qbittorrent | grep -i "temporary password"
# or: python3 ~/git/torrent_finder/deploy/raspberry-pi/qbt_password.py
#     and paste the line under [Preferences]
```

### Prowlarr (`http://<host>:9696`)

1. *Settings → General → Security*: copy the API key into `PROWLARR_API_KEY`.
2. *Indexers → Add Indexer → TorrentDay*, authenticate with your account cookie.
3. Run one search from Prowlarr's own *Search* tab before trusting it from code.
4. Recreate the finder so it picks up the key: `docker compose up -d finder`.

### torrent-finder (`http://<host>:8010`, `media.bjorngreen.se`)

Nothing to configure in a UI; everything is environment. Confirm the routing
resolved against the real qBittorrent:

```bash
docker compose logs finder | grep 'qBittorrent category'
# movie -> qBittorrent category 'movies' at /data/movies
# tv    -> qBittorrent category 'tv' at /data/tv
# other -> qBittorrent category 'other' at /data/other
```

`/downloads/movies` in that output means the `*_SAVE_PATH` variables never
reached the container. Search behaviour: an empty search box browses the ticked
categories instead of matching a name. Deleting from the downloads drawer
deletes the files, which for a finished torrent is the copy Plex is serving.

### Plex (`http://<host>:32400/web`)

First run only: uncomment `PLEX_CLAIM`, put a fresh token from
https://plex.tv/claim in `.env` (it expires in 4 minutes), start Plex, then
remove it again. Two libraries, each spanning both disks:

| Library | Folders |
| --- | --- |
| Movies | `/active/movies`, `/archive/movies` |
| TV Shows | `/active/tv`, `/archive/tv` |

Both mounts are read-only, so Plex's own *delete media* does nothing. Deleting
is the finder's job, via qBittorrent, which holds `/data` read-write. `other/`
is deliberately outside any library.

**Hardware transcoding (Quick Sync)** — mini PC only; the Pi has no hardware
Plex can use, so on the `pi` branch expect direct play and stop here.

The N95's iGPU does H.264 and HEVC in hardware. `hw/x86.yaml` passes `/dev/dri`
and joins the host's `render` group via `RENDER_GID` (§3); the remaining
steps are *Settings → Transcoder →* tick **Use hardware acceleration when
available** (and HW-accelerated video encoding), which needs **Plex Pass**.
Verify a transcode is actually offloaded:

```bash
ls -l /dev/dri/renderD128                            # host: device exists
docker compose exec plex ls -l /dev/dri/renderD128    # container: readable
sudo apt install intel-gpu-tools && sudo intel_gpu_top   # Video engine busy
```

The dashboard marks an offloaded stream `(hw)`. `permission denied` on
`renderD128` inside the container means `RENDER_GID` does not match
`getent group render` on this host. If `/dev/dri` is missing *inside* the
container while present on the host, `COMPOSE_FILE` is not selecting
`hw/x86.yaml`, or something passed `-f` (§3).

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

The app repo needs nothing but a `Dockerfile`; the stack owns how it is wired in.

1. Clone it into `~/git/<new-app>`.
2. Write `~/stack/apps/<new-app>.yaml` from the skeleton below — one service,
   `container_name` set so the tunnel URL is predictable, state under
   `/srv/appdata/<new-app>`, and a host port from §4 only if you want LAN access.
3. Add it to `~/stack/compose.yaml` under `include:`.
4. Add a line to `~/stack/autodeploy.conf`.
5. Add any secrets to `~/stack/.env` **and** `.env.example`, referenced with
   `${VAR:?set in ~/stack/.env}` so a missing value fails loudly at start.
6. Add the public hostname in Cloudflare (§9) plus an Access policy.
7. Update the port registry in §4 and the table at the top of this file.

Whatever compose file the app repo ships is ignored here, so it may keep its own
`cloudflared`, its own `./data` volume and its own ports for standalone use.

```yaml
# ~/stack/apps/<new-app>.yaml
services:
  <new-app>:
    build: ../../git/<new-app>
    container_name: <new-app>
    environment:
      TZ: "${TZ:-Europe/Stockholm}"
      SOME_SECRET: "${NEWAPP_SECRET:?set in ~/stack/.env}"
    volumes:
      - /srv/appdata/<new-app>:/data   # internal SSD, outside the git checkout
    # Media-adjacent apps mount /mnt/active or /mnt/archive read-only instead;
    # only qBittorrent gets /mnt/active read-write.
    # ports:                           # only if you want LAN access
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
`/home/bjorngreen`; change both if the Ubuntu user is not `bjorngreen`.

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
  ever edit code directly on the server. If you do, switch it to `git pull
  --ff-only` and let it fail loudly instead.

Not covered by autodeploy, deliberately: `~/stack` itself. Infrastructure
changes restart real services, so pull those by hand (§12).

## 12. Routine operations

Always from `~/stack` and always without `-f`, so `.env`'s `COMPOSE_FILE` picks
up the hardware overlay (§3).

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
df -h / /mnt/active /mnt/archive
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
  -C / srv/appdata/qbittorrent/qBittorrent \
       srv/appdata/prowlarr/config.xml \
       srv/appdata/plex/config/"Library/Application Support/Plex Media Server/Preferences.xml" \
       srv/appdata/matte-vm \
  -C "$HOME" stack/.env
```

Copy it off the box. Everything irreplaceable now lives under `/srv/appdata` on
one disk, which is convenient for backups and the reason that disk is the one to
mirror: media is re-downloadable, but `.env`, the Plex library database and
matte-vm's SQLite file are not.

## 13. Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `services.cloudflared conflicts with imported resource` | an included app compose file defines its own cloudflared | include a stack-owned fragment under `apps/` instead (§10) |
| a container's config directory is empty and never persists | Docker installed from **snap**; confinement blocks binds outside `/home` | `snap remove --purge docker`, reinstall from Docker's repo (§2) |
| plex: `permission denied` opening `/dev/dri/renderD128` | `RENDER_GID` ≠ the host's render gid | `getent group render \| cut -d: -f3`, fix `.env`, recreate plex |
| transcodes stay software-only despite `/dev/dri` | Plex Pass missing, or the Transcoder setting not ticked | §8 |
| `/dev/dri` present on the host, absent in the container | `docker compose` was given `-f`, or `COMPOSE_FILE` omits `hw/x86.yaml` | run bare `docker compose` from `~/stack` (§3) |
| `/` is ~100 GB on a 500 GB disk | Ubuntu's guided LVM left the VG unallocated | `lvextend -l +100%FREE` + `resize2fs` (§1) |
| a published port is reachable despite a ufw rule | Docker's rules sit ahead of ufw's chains | delete the `ports:` entry; ufw cannot close it (§2) |
| every container restarted overnight | `unattended-upgrades` updated docker-ce | `apt-mark hold` the docker packages (§2) |
| `conflicting options: port publishing and the container type network mode` | a `ports:` block on a service using `network_mode: service:gluetun` | move the port to gluetun's `ports:` |
| `port is already allocated` | host port collision | pick a free host port, update §4 |
| finder: `Cannot reach qBittorrent at http://qbittorrent:8080` | namespace tenants have no DNS name | use `http://gluetun:8080` |
| finder: `qBittorrent login failed` | temporary password rotated on restart | permanent password + subnet whitelist (§8) |
| `502`/`530` from a public hostname | cloudflared cannot resolve or reach the service | service name and **container** port; is it in the same project? |
| Cloudflare hostname 404s all assets | a Path prefix was set | leave Path empty, use a subdomain |
| `docker compose exec qbittorrent curl ifconfig.me` shows the ISP IP | namespace sharing not in effect | check `network_mode`, recreate |
| Ubuntu stuck at boot with no network | a USB SSD did not mount | `nofail` in `/etc/fstab` (§1) |
| that same command hangs | gluetun killswitch, usually DNS | `docker compose logs gluetun` |
| Plex shows half-finished files | downloading straight into a library | set the incomplete path (§8) |
| finished torrents take minutes to "move" | a category path escaped the `/data` mount | every category lives under `/data`, one disk (§1, §8) |
| Plex cannot delete a file | `/active` and `/archive` are mounted read-only | intended; delete through the finder (§8) |
| moved a folder to archive and seeding stopped | qBittorrent only mounts `/mnt/active` | expected — the trade described in §1 |
| finder refuses a download for low space | free-space guard reading the wrong disk | default save path must be on ssd2 (§8) |
| finder container exits at start | a `${VAR:?...}` has no value | fill it in `~/stack/.env` |
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
- **`ufw` is not part of the posture.** Docker's rules run ahead of ufw's, so
  enabling it does not close a published port (§2); what keeps the LAN-only
  services LAN-only is that no router port is forwarded. Removing a `ports:`
  entry is the real control.
- Ubuntu's `unattended-upgrades` keeps the host patched, which is worth having;
  if you `apt-mark hold` the docker packages to stop surprise restarts (§2),
  updating them stays your job.
- Rotate anything that has ever been pasted into a chat, an issue or a paste
  site: NordVPN service credentials, the tunnel token, `UI_PASSWORD`.

## 15. Migrating an existing setup

Two separate jobs, and they can happen months apart: restructuring the disks
into the layout in §1, and moving the box. Do the restructure first, on the Pi,
so the new layout is proven before new hardware is in the way.

### 15a. Run this on the Pi now — the `pi` branch

```bash
cd ~/stack && git fetch && git checkout pi
```

The `pi` branch differs from `main` in this file only — the hardware, dependency,
setup and troubleshooting sections — plus the default `COMPOSE_FILE` in
`.env.example`. Every compose file is byte-identical, because the real hardware
difference is `hw/pi.yaml`, selected by one line in `.env` (§3):

```dotenv
COMPOSE_FILE=compose.yaml:hw/pi.yaml
```

`RENDER_GID` stays blank there; nothing on the Pi reads it.

### 15b. Restructuring the disks

Nothing large moves. The drive that already holds the torrent folders becomes
`active`, so the bulk of the data stays exactly where it is and only directory
names and mountpoints change — both instant, both within one filesystem.

Point the mountpoints at their new roles in `/etc/fstab` (§1), remount, then:

```bash
cd ~/stack && docker compose down

# active  = the old /mnt/ssd2: already has incomplete/ and the torrent folders
mv /mnt/active/share/torrents/movies      /mnt/active/movies
mv /mnt/active/share/torrents/tv          /mnt/active/tv
mv /mnt/active/share/torrents/incomplete  /mnt/active/incomplete
rmdir /mnt/active/share/torrents /mnt/active/share

# archive = the old /mnt/ssd: the hand-managed library
mv /mnt/archive/media/movies  /mnt/archive/movies
mv /mnt/archive/media/tv      /mnt/archive/tv

# container state off both media drives and onto its own place
mv /mnt/archive/qbittorrent-config  /srv/appdata/qbittorrent
mv /mnt/archive/prowlarr-config     /srv/appdata/prowlarr
mv /mnt/archive/gluetun             /srv/appdata/gluetun
mv /mnt/active/plex/config          /srv/appdata/plex/config
cp -r ~/git/matte-vm/data/.         /srv/appdata/matte-vm/

# the one real copy: `other` was on the wrong disk (see below)
mv /mnt/archive/media/other/* /mnt/active/other/ && rmdir /mnt/archive/media/other
sudo chown -R 1000:1000 /srv/appdata /mnt/active /mnt/archive
```

That last one is a genuine cross-disk copy, and it is fixing a bug rather than
creating one: `other` used to save to `/downloads/other` on the *library* disk
while qBittorrent's incomplete directory sat on the *torrent* disk, so every
completed `other` download was already being copied between disks — the exact
thing §1 forbids, on the one category nobody watches. Under `/data` it cannot
happen again.

### 15c. Repointing qBittorrent's existing torrents

The container-side paths changed (`/auto_movies` → `/data/movies`), so running
torrents will report missing files until they are told where the data went.
qBittorrent has no bulk rewrite, but per-category is close enough:

1. Start the stack, open the WebUI, click a category in the sidebar.
2. Select all (Ctrl-A) → right-click → **Set location** → the new path.
3. It rechecks — fast, since the files are present and unmoved.

Repeat for each category. If you would rather not touch the session at all, add
compatibility mounts to `media/compose.yaml` so both path spellings resolve,
migrate at leisure, then delete them:

```yaml
      - /mnt/active/movies:/auto_movies
      - /mnt/active/tv:/auto_tv
      - /mnt/active/incomplete:/auto_incomplete
```

The alternative is legitimate: let the Pi seed the old layout until it drains,
and start the new box with a fresh session. Finished files are already in the
library; only in-flight torrents are lost.

### 15d. Moving to the mini PC

By this point the layout is proven and the move is mostly cabling.

```bash
# on the Pi
cd ~/stack && docker compose down
tar czf ~/state.tgz -C / srv/appdata -C "$HOME" stack/.env
```

Then on the mini PC, after §1–§3 and before first start (§7 step 5):

| Carry over | How |
| --- | --- |
| `/srv/appdata` | untar; `chown -R 1000:1000` afterwards |
| `~/stack/.env` | untar, then set `COMPOSE_FILE=compose.yaml:hw/x86.yaml` and fill `RENDER_GID` |
| both USB drives | unplug, replug, fix the UUIDs in `/etc/fstab` |
| `~/git/*` | plain `git clone`; nothing in a checkout is state any more |

Four things to know about the moved state:

- **Plex's library survives the architecture change.** The database is portable,
  and the library *paths* that matter are container-side (`/active/movies`,
  `/archive/movies`), so they do not change with the hardware. No full rescan.
- **Keep a `plex.tv/claim` token ready** in case the server shows as unclaimed;
  usually the machine identity in the config carries over and it does not.
- **Copy qBittorrent's config only while the container is stopped**, or it will
  be overwritten on exit. Its session survives, because by now the container
  paths are already the new ones.
- **Prowlarr's TorrentDay cookie may have expired.** Re-authenticating the
  indexer is quicker than debugging an empty search.

When the Pi is retired, the branch goes with it:

```bash
git branch -d pi && git push origin --delete pi
```

Two leftovers in the app repos, since the stack now owns this documentation:
`~/git/torrent_finder/deploy/HOMELAB.md` is a stale copy of this file and can go,
and `deploy/raspberry-pi/` is now just misnamed (`qbt_password.py` in it still
works — §8 links to it).
