# homelab runbook

Everything needed to build this server from a blank disk, plus how to change or
extend it afterwards. The hardware is a BOSGAME mini PC (Intel N95) running
Ubuntu Server 24.04; a Raspberry Pi is still a supported target on the `pi`
branch.

**What runs here**

| Stack | Services | Reachable at | Configured in |
| --- | --- | --- | --- |
| media | gluetun (VPN), qBittorrent, Prowlarr, Plex | LAN ports | [media/README.md](media/README.md) |
| automation | Radarr, Sonarr, Bazarr | LAN ports only — admin tools | [media/README.md](media/README.md) |
| home | Home Assistant, Mosquitto, Zigbee2MQTT | LAN ports only | [home/README.md](home/README.md) |
| games | ES-DE, RetroArch | the TV — HDMI out, no port at all | [games/README.md](games/README.md) |
| matte-vm | FastAPI + SQLite + PWA | its own Cloudflare hostname | [apps/README.md](apps/README.md) |
| jellyseerr | request front-end for Plex | its own Cloudflare hostname | [apps/README.md](apps/README.md) |
| kometa | Plex collections and posters | nothing — scheduled job, no UI | [apps/README.md](apps/README.md) |
| shared | cloudflared — **one** tunnel for the whole box | — | this file, §9 |

**This file is the machine.** Disks, packages, the repo layout, `.env`, the
first-run order, the tunnel, deployment and routine operations — everything you
do once, to a new box. What happens *inside* each application afterwards lives
with the compose fragment that defines it, in the four documents above.

**Design decisions, so the layout is not mysterious later**

- **One Compose project, assembled with `include:`.** One `docker compose up -d`
  brings up the machine, and everything shares one network, so containers reach
  each other by service name with no external network to maintain.
- **No git submodules.** Each app keeps its own repo and its own compose file,
  independently deployable; `~/stack` only points at them. Submodules would add
  a commit-the-pointer step to every app change and buy nothing on a single box.
- **One tunnel, one cloudflared.** Adding a hostname is then always the same
  action in the same place. Apps must not ship their own cloudflared.
- **App code and app data never live on the boot disk.** Container config and
  media go on the SSDs; the system disk holds the OS and the git checkouts.
- **Each stack documents itself.** A fragment and its runbook sit in the same
  directory, so changing one without the other is conspicuous.

---

## 1. Hardware, OS and disks

- BOSGAME mini PC, Intel N95 (Alder Lake-N, x86_64), Ubuntu Server 24.04 LTS.
  The Raspberry Pi is still a supported target — see the `pi` branch.
- Wired ethernet. A static DHCP lease makes LAN URLs stable.

Three disks, three jobs:

| Mount | Disk | Job |
| --- | --- | --- |
| `/srv/appdata` | internal 500 GB SSD | all container state; mostly empty on purpose |
| `/mnt/active` | **1 TB USB** (the old `/mnt/ssd2`) | everything in flight and everything currently seeding |
| `/mnt/archive` | **500 GB USB** (the old `/mnt/ssd`) | content aged off active; read-only to every container |

```
/srv/appdata/          internal 500 GB SSD — all container state, grouped to
  media/               mirror the compose fragments: one directory per fragment
    gluetun/  qbittorrent/  prowlarr/  radarr/  sonarr/  bazarr/
    plex/{config,transcode}/
  home/
    homeassistant/  mosquitto/  zigbee2mqtt/
  apps/
    jellyseerr/  kometa/  matte-vm/
  games/
    es-de/               settings, themes, gamelists, scraped artwork
    retroarch/           cores, save files, save states

/mnt/active/           1 TB USB — everything torrents touch
  incomplete/          qBittorrent's default save path AND its incomplete path
  downloads/movies/    qBittorrent's targets; torrents keep seeding from here
  downloads/tv/
  library/movies/      Radarr's and Sonarr's output; what Plex reads
  library/tv/
  books/               manual grabs — ebooks, audiobooks, magazines
  other/               manual grabs — anything else
  games/roms/<system>/ the tree ES-DE reads; hardlinked out of other/
  games/bios/          BIOS images for the cores that need them

/mnt/archive/          500 GB USB — content moved off active once it fills
  movies/  tv/  ...    read-only; mirrors whichever active folders you move
```

**Why `downloads/` and `library/` both exist — nothing is duplicated.** They are
two *names* for one set of blocks, via hardlinks, so a film seeding under its
original release name and the same film named tidily for Plex cost the size
once. [media/README.md](media/README.md#how-the-storage-model-works) explains the
model, why Radarr must never move a file, and what it costs when a hardlink
silently degrades into a copy.

The only rule that matters at this level: **everything torrents touch is one
filesystem**, `/mnt/active`, mounted at the same container path `/data`
everywhere. Split it across two disks and every completion becomes a copy.

**Why state is internal and media is not.** Plex's library is a SQLite database
bound by random writes, so its location is the one you can actually feel; media
is sequential, re-downloadable, and far too big for 500 GB. All three disks are
SSDs, so the mountpoints name each disk's *job* rather than its bus.

**Why one disk holds everything torrents touch.** qBittorrent gets a single
mount — `/mnt/active` as `/data` — so `incomplete/`, `movies/`, `tv/` and
`other/` are unavoidably on one filesystem. Completing a torrent is then a
rename instead of a copy, and it cannot regress into a copy later by someone
pointing one category at the other disk — and the same one filesystem is what
lets Radarr and Sonarr import by hardlink instead of copying
([media/README.md](media/README.md)).

That last point is why downloads stay on a USB drive rather than following the
rest of the state onto the fast internal SSD: put `incomplete/` there and every
completion becomes a cross-disk copy, while `MIN_FREE_SPACE_GB` starts guarding
500 GB of system disk as the media drive silently fills.

**The bigger drive is `active`, and that is not arbitrary.** It follows from the
hardlinks. A 20 GB film seeding in `downloads/movies/` and named properly in
`library/movies/` is *one* set of bytes with two names, so it costs 20 GB, not
40 — which means active's real occupancy is the unique size of everything you
currently seed, and it holds far more than the directory listing suggests.

Moving something to archive is the single operation that breaks that. Archive is
a different filesystem, so the move is a real copy, the hardlink is severed, and
qBittorrent loses the file:

```bash
mv /mnt/active/library/movies/"Some Film (2019)" /mnt/archive/movies/
```

Plex finds it again, because both directories are in the same library
([media/README.md](media/README.md)).
qBittorrent does not, so that torrent stops seeding — the intended trade, and the
reason this is a deliberate manual step rather than a script. **Free space on
active is therefore exactly "how long until I have to shuffle files by hand", so
it goes to the larger disk.** Archive being smaller costs nothing: it holds cold
files that are already downloaded and no longer seeding.

If you do not mind losing seeds, none of this is delicate — moving a folder to
archive is just a copy, and content can be shuffled between the two whenever it
suits. Radarr and Sonarr only manage the `library/` root on active, so anything
in archive is simply files that Plex reads and nothing else tracks.

Roughly 880 GiB usable on active after ext4's overhead, 440 GiB on archive, so
about 1.3 TB of unique video in total — and whatever sits on active is
simultaneously seeding at no extra cost. If you want the last few GB, ext4
reserves 5% for root by default, which is pointless on a media drive:

```bash
sudo tune2fs -m 1 /dev/sdXn      # ~45 GB back on the 1 TB, ~22 GB on the 500 GB
```

**The internal 500 GB is meant to stay mostly empty.** Ubuntu, Docker's images
and `/srv/appdata` together come to well under 100 GB, and the spare is
deliberate headroom: Plex's metadata and thumbnails grow with the library, and
`plex/transcode` is scratch that spikes. Resist the temptation to put media on
it — `incomplete/` there would make every completion a cross-disk copy, and a
library folder there could not be hardlinked from `downloads/`. Backup tarballs
(§12) staged before they leave the box are the one other reasonable use.

Two disks is also the point at which a union filesystem (mergerfs) starts paying
for itself, presenting both as one tree so nothing ever has to be moved by hand.
It is not worth its mount-order failure modes at this size.

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
| `/dev/dri/renderD128` | Plex Quick Sync ([media/README.md](media/README.md)) | present once `i915` loads |

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

**The NordLynx private key.** gluetun runs NordVPN over WireGuard, which
authenticates with a key instead of a username and password. Nord does not put it
in the dashboard, but their API will hand it over:

```bash
# Nord dashboard → NordVPN → Manual setup → generate an access token
curl -s -u token:<ACCESS_TOKEN> https://api.nordvpn.com/v1/users/services/credentials \
  | jq -r .nordlynx_private_key
```

That value goes in `WIREGUARD_PRIVATE_KEY`. If the endpoint has moved, gluetun's
own NordVPN documentation is the place to look — it tracks this. Failing that,
Nord's Linux client stores the same key once connected, readable with
`sudo wg show nordlynx private-key`.

Confirm the tunnel is actually up before anything else starts (§7):

```bash
docker compose logs gluetun | grep -i 'public ip'
# want: "Public IP address is ... (Sweden ...)"
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
| NordVPN **NordLynx private key** | see below. *Not* your login, and not the OpenVPN service credentials either. |
| Cloudflare account + `bjorngreen.se` on it | dash.cloudflare.com |
| Cloudflare tunnel token | Zero Trust → Networks → Tunnels → Create → Docker → the `TUNNEL_TOKEN` value |
| TorrentDay account | for the Prowlarr indexer (cookie auth) |
| Plex claim token | https://plex.tv/claim — valid 4 minutes, first run only |
| Plex Pass | only for hardware transcoding; without it Plex transcodes in software |

## 3. Repository layout

```
~/stack/                          THIS repo — how the machine fits together
  README.md                       this file: the machine, start to finish
  compose.yaml                    include: list + cloudflared
  media/compose.yaml              gluetun, qbittorrent, prowlarr, plex
  media/arr.yaml                  radarr, sonarr, bazarr
  media/README.md                 configuring all six, + the storage model
  home/compose.yaml               homeassistant, mosquitto, zigbee2mqtt
  home/README.md                  configuring all three
  games/compose.yaml              es-de, retroarch — the TV
  games/README.md                 ROMs, Bluetooth pads, the display
  apps/matte-vm.yaml              one fragment per standalone app
  apps/jellyseerr.yaml            third-party image, same shape
  apps/kometa.yaml                scheduled job: no port, no hostname
  apps/README.md                  configuring those three, + adding a new app
  hw/x86.yaml                     the only per-hardware overlay, picked in .env
  bin/bootstrap                   provision a fresh box up to `up -d` (§7)
  bin/autodeploy                  poll git, redeploy what changed
  bin/roms-import                 hardlink downloaded ROMs into ES-DE's tree
  bin/games-link-emulators        let ES-DE find RetroArch across containers
  autodeploy.conf                 repo -> service map
  systemd/autodeploy.{service,timer}   installed into /etc (§11)
  .env                            ALL secrets, gitignored
  .env.example                    same keys, no values, committed
  .gitignore
~/git/matte-vm/                   app repo: Dockerfile + its own compose file
~/git/<next-app>/                 same shape
/srv/appdata/<group>/<service>/   container state, internal SSD; <group> is the
                                  compose fragment's own directory (§1)
/mnt/active/, /mnt/archive/       bulk media, USB drives (§1)
```

**Every stack directory holds its own README.** `media/`, `home/`, `apps/` and
`games/` each document the applications their fragment defines — the settings,
the first login, and the traps specific to those services. This file stops at
the point the containers are running.

Why apps stay outside `~/stack`: each one keeps a compose file that works on its
own (`cd ~/git/matte-vm && docker compose up`), which is what makes it testable
in isolation, while `~/stack` composes them into the real machine.

**The stack does not read those app compose files.** Each app gets a fragment
here that only `build:`s from `~/git/<app>`. That is the pattern `media/` already
used for `finder` before it was retired, generalised — because an included app
file must not define
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

Exactly one thing genuinely differs between the mini PC and the Pi — Plex's Quick
Sync device — so it lives in [`hw/x86.yaml`](hw) rather than in the files above,
and `.env` picks it up:

```dotenv
COMPOSE_FILE=compose.yaml:hw/x86.yaml    # the Pi uses just compose.yaml
```

There used to be a `hw/pi.yaml` as well, holding an `AES-128-GCM` cipher pin for
a CPU without AES-NI. Moving to WireGuard deleted it: WireGuard has a single
cipher suite, ChaCha20-Poly1305, which is not configurable and happens to be the
faster choice on hardware without AES acceleration anyway. The Pi therefore needs
no overlay at all. Add the file back if a Pi-only override ever appears.

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
| 8090 | frontend (nginx) | container side is 80 |
| 7878 | Radarr | LAN only, never tunnelled |
| 8989 | Sonarr | LAN only, never tunnelled |
| 6767 | Bazarr | LAN only, never tunnelled |
| 5055 | Jellyseerr | its own default |
| 1883 | Mosquitto (MQTT) | published so host-networked HA can reach it |
| 9001 | Mosquitto (websockets) | |
| 8082 | Zigbee2MQTT frontend | container side is **8080**; deCONZ wants this host port too — see [home/README.md](home/README.md) |
| 8123 | Home Assistant | `network_mode: host`, so no `ports:` entry |
| 32400 | Plex | `network_mode: host` |
| — | ES-DE, RetroArch | no port by design: the output is HDMI ([games/README.md](games/README.md)) |

Container ports never collide, only host ports. An app reachable solely through
the tunnel needs no `ports:` entry at all, and a batch job like `kometa` needs
none either — it never listens.

## 5. The rule that breaks things first

`qbittorrent` and `prowlarr` use `network_mode: "service:gluetun"`, so they have
**no network identity of their own**. The address depends on which side you ask
from:

| From | To | Address |
| --- | --- | --- |
| a bridge service — Radarr, Sonarr, Bazarr, Jellyseerr | qBittorrent | `http://gluetun:8080` |
| a bridge service | Prowlarr | `http://gluetun:9696` |
| **Prowlarr** (inside the namespace) | **qBittorrent** | **`http://localhost:8080`** |
| anywhere | Radarr, Sonarr, Bazarr | their own names — ordinary bridge services |

- `http://qbittorrent:8080` and `http://prowlarr:9696` never resolve from
  anywhere. Neither name exists.
- Neither service may have a `ports:` block — all their ports go on gluetun.
- Restarting or recreating gluetun restarts both and interrupts every active
  torrent, which is why a routine app deploy must never touch it (§11).

[media/README.md](media/README.md#the-rule-that-breaks-things-first) has the full
reasoning, including why `gluetun` does not work from *inside* the namespace.

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
| `WIREGUARD_PRIVATE_KEY` | NordVPN | gluetun |
| `CLOUDFLARE_TUNNEL_TOKEN` | Cloudflare | cloudflared |
| qBittorrent's WebUI login | qBittorrent's API | Radarr, Sonarr and Prowlarr, each in its own UI |
| Prowlarr's / Radarr's / Sonarr's API keys | each other | entered in each app's UI, never in `.env` |

## 7. First-time setup, in order

[`bin/bootstrap`](bin/bootstrap) does steps 1–4 below, plus §11's timer. Three
commands come first, because cloning the repo that holds the script needs git:

```bash
sudo apt update && sudo apt install -y git
git clone git@github.com:<you>/server_stack.git ~/stack
~/stack/bin/bootstrap
```

It is idempotent — re-running it is the intended way to audit a box, and each
step either verifies what is already true or skips itself. It detects Pi vs mini
PC and writes the matching `COMPOSE_FILE` (§3), fills `RENDER_GID` from the
host's render group, refuses to continue on Debian's `docker.io` (§2), creates
the directory tree, and finishes by naming every `.env` value still empty.

**Three things it will not do**, all for the same reason — the cost of guessing
wrong is higher than the keystrokes saved:

- **Touch `/etc/fstab`.** UUIDs are per-disk, and a wrong line leaves a headless
  box in emergency mode with no network (§1). It prints the lines with the
  candidate disks listed, and stops until the mounts are real.
- **Create the directory tree before the mounts exist.** `mkdir` onto an
  unmounted `/srv/appdata` *succeeds*, on the boot disk, silently — the failure
  in §13's second row. So the mount check gates the tree, which is why the
  script exits rather than carrying on.
- **Invent secrets.** It reports which are missing and where each comes from.

The rest of this section is what the script does, in case it fails or you are on
something it does not cover:

```bash
# 1. clone
mkdir -p ~/git && cd ~/git
git clone git@github.com:<you>/matte-vm.git
# This repo. The GitHub repo is `server_stack`, but the checkout must be
# ~/stack: bin/autodeploy defaults to $HOME/stack and the systemd units name
# /home/bjorngreen/stack. Hence the explicit target directory.
cd ~ && git clone git@github.com:<you>/server_stack.git stack

# 2. secrets
cp ~/stack/.env.example ~/stack/.env && chmod 600 ~/stack/.env && nano ~/stack/.env

# 3. data directories, owned by the container user
#    internal SSD: container state.  USB drives: bulk media only.
sudo mkdir -p /srv/appdata/media/{gluetun,qbittorrent,prowlarr,radarr,sonarr,bazarr,plex/{config,transcode}}
sudo mkdir -p /srv/appdata/home/{homeassistant,mosquitto,zigbee2mqtt}
sudo mkdir -p /srv/appdata/apps/{jellyseerr,kometa,matte-vm}
sudo mkdir -p /srv/appdata/games/{es-de,retroarch}
sudo mkdir -p /mnt/active/{incomplete,downloads/{movies,tv},library/{movies,tv}} \
              /mnt/active/{books/{audiobooks,ebooks,magazines},other} \
              /mnt/active/games/{roms,bios} \
              /mnt/archive/{movies,tv}
sudo chown -R 1000:1000 /srv/appdata /mnt/active /mnt/archive
# mosquitto is the one exception: its image runs as uid 1883, not 1000, and
# cannot write persistence or logs under 1000 (home/README.md).
sudo chown -R 1883:1883 /srv/appdata/home/mosquitto

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

## 8. Per-service configuration

Everything that happens *after* `docker compose up -d` — the settings inside each
application, and the traps specific to it — lives with the compose fragment that
defines it:

| Stack | Document | Covers |
| --- | --- | --- |
| media | [media/README.md](media/README.md) | qBittorrent, Prowlarr, Radarr, Sonarr, Bazarr, Plex, and the hardlink storage model |
| home | [home/README.md](home/README.md) | Home Assistant, Mosquitto, Zigbee2MQTT |
| apps | [apps/README.md](apps/README.md) | Jellyseerr, Kometa, matte-vm, and how to add a new app |
| games | [games/README.md](games/README.md) | ES-DE, RetroArch, ROM import, Bluetooth controllers |

**Start with [media/README.md](media/README.md).** It holds the first-login table
for the \*arr apps — none of them ships a default password, and only qBittorrent
prints one — plus the two rules the rest of the stack depends on.

Order that works: qBittorrent (set a password, point it at `/data`), Prowlarr
(indexers, then push them to the \*arrs), Radarr and Sonarr (download client,
root folders, quality profiles), then Plex, then the rest.

## 9. Cloudflare

One tunnel, one cloudflared, hostnames added in the dashboard — the token-based
tunnel is **remotely managed**, so a local `config.yml` is ignored.

*Zero Trust → Networks → Tunnels → your tunnel → Public hostnames → Add*:

| Subdomain | Domain | Path | Type | URL |
| --- | --- | --- | --- | --- |
| (matte) | `bjorngreen.se` | *(empty)* | HTTP | `app:8000` |
| `requests` | `bjorngreen.se` | *(empty)* | HTTP | `jellyseerr:5055` |

**Radarr, Sonarr, Bazarr and qBittorrent get no hostname, on purpose.** They can
rewrite or delete your library and they hold tracker credentials, and Jellyseerr
already covers the only thing anyone needs to do remotely. Reach them on the LAN
or through whatever you already use to reach the box's shell. If `media.` still
points at the retired finder, delete that hostname and its Access application.

The URL is `<service name>:<container port>` — the container port, not the host
port, because cloudflared talks over the shared docker network. Leave **Path**
empty and give each app its own hostname: these apps serve assets from absolute
paths (`/static/app.js`), so a path prefix loads the page and 404s everything
else.

**Then put Access in front of anything that can act on your box.** *Zero Trust →
Access → Applications → Add a self-hosted application*, hostname
`requests.bjorngreen.se`, policy **Allow → Emails → your address** (one-time PIN
needs no extra setup) — but read the Jellyseerr caveat below before applying it,
because it is the one hostname where Access may be the wrong call.

**Jellyseerr is the one case where Access may be wrong.** It signs people in with
Plex OAuth, so it already has real per-user auth, and it exists for other people
to use. An Access policy in front means adding every requester's email address to
Cloudflare as well, and they hit two unrelated login screens. If it is only ever
you, keep Access. If anyone else requests, drop Access for that hostname and let
Jellyseerr's own Plex sign-in be the lock — it can accept nothing more dangerous
than a request. Set *Settings → General → Application URL* to the public
hostname either way, or Plex's OAuth redirect and the notification links break.

**If `cloudflared` restart-loops**, it is exiting rather than retrying, and that
almost always means the token — a connectivity problem makes it log failures and
keep trying instead. The log says which:

```bash
cd ~/stack && docker compose logs --tail=30 cloudflared
```

| Log says | Cause |
| --- | --- |
| *Provided Tunnel token is not valid* | truncated on paste, or a stray newline. The token is one long base64 string with no spaces |
| *Unauthorized* / *tunnel not found* | the tunnel was deleted in the dashboard, or the token belongs to a different one |
| repeated *Couldn't connect to Cloudflare edge*, no exit | genuinely the network — this one does not restart-loop |

Check the value without printing it:

```bash
cd ~/stack
tok=$(grep '^CLOUDFLARE_TUNNEL_TOKEN=' .env | cut -d= -f2-)
echo "length: ${#tok}"        # a real token is roughly 180-220 characters
[[ "$tok" =~ ^[A-Za-z0-9+/=_-]+$ ]] && echo "charset OK" || echo "contains a space, quote or newline"
```

Short means truncated on paste. A charset failure means either surrounding quotes
or — the usual one — the whole `docker run …` command was copied instead of just
its last argument. Editors that hard-wrap will also insert a newline into a
200-character line; paste it with `wrap` off, or write the line with `printf`
rather than an editor.

Re-copy it from *Zero Trust → Networks → Tunnels → your tunnel → Configure →
Docker*: the token is the long string at the end of the command shown there, not
the command.

Then **recreate, do not restart**:

```bash
docker compose up -d cloudflared
```

`docker compose restart` reuses the existing container, which still holds the old
value — environment is baked in at creation. This applies to every `.env` change,
not just this one, and is a reliable way to lose an hour.

**This blocks remote access only.** Everything on the box is still reachable on
the LAN at its own port, Jellyseerr included, and Plex sign-in works there — the
tunnel is how other people reach it, not how it works. Do not wait on cloudflared
to finish setting anything up.

```bash
curl -sI https://requests.bjorngreen.se | head -1
# 401 = the app answered and refused for lack of credentials (healthy)
# 502/530 = cloudflared could not reach the service — name or port wrong
# Cloudflare login page = Access is in front (expected once §9 is done)
```

## 10. Adding a new app

The three shapes an app can take — built from a repo, shipped as an image, or a
scheduled job with no UI — and a skeleton fragment for each are in
[apps/README.md](apps/README.md#adding-a-new-app), next to the fragments they
describe.

Whatever shape it is, four files in *this* directory change:

1. `apps/<new-app>.yaml` — the fragment itself.
2. `compose.yaml` — add it under `include:`.
3. `autodeploy.conf` — one line, **only** if it builds from a repo you push to.
4. `.env` **and** `.env.example` — any secret, referenced as
   `${VAR:?set in ~/stack/.env}` so a missing value fails loudly at start.

Then update the port registry in §4 and the table at the top of this file, and
add the hostname in Cloudflare (§9) if it needs one.

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

`bin/bootstrap` installs and enables these, but only when the user really is
`bjorngreen` at `~/stack` — anywhere else it skips the step and says so, because
enabling a timer against the wrong paths just fails every five minutes. By hand:

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

- **`--no-deps <service>`** — without it a rebuild can recreate gluetun,
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
docker compose up -d --build --no-deps app

# infrastructure change (edited compose.yaml or .env) — `up -d`, never `restart`:
# environment is baked in when a container is created, so restart keeps the old
# values and the edit looks like it did nothing
git pull && docker compose config -q && docker compose up -d

# base image updates — media stack only, on purpose
docker compose pull qbittorrent prowlarr radarr sonarr bazarr plex cloudflared
docker compose up -d qbittorrent prowlarr radarr sonarr bazarr plex cloudflared

# home automation — separately, because HA migrates its config forward on start
docker compose pull homeassistant mosquitto zigbee2mqtt
docker compose up -d homeassistant mosquitto zigbee2mqtt
docker image prune -f

# gluetun / VPN change: recreates its tenants, interrupts torrents
docker compose pull gluetun && docker compose up -d gluetun

# recreate everything — see below for why plain `up -d` often does nothing
docker compose up -d --force-recreate

# state
docker compose ps
docker compose logs -f --tail=100 radarr
df -h / /mnt/active /mnt/archive
docker system df
```

**When `up -d` reports nothing but the network.** `docker compose up -d` is
idempotent: it recreates a container only when its definition or image has
actually changed, so on an unchanged stack it leaves everything running and says
so. If you want new containers regardless, that is what `--force-recreate` is
for, and there is a ladder of increasing severity:

| Command | Does |
| --- | --- |
| `docker compose up -d` | recreates only what changed |
| `docker compose up -d --force-recreate` | recreates every container from the current config |
| `docker compose up -d --force-recreate --build` | the same, rebuilding images from the `build:` contexts too |
| `docker compose pull && docker compose up -d --force-recreate` | also takes new upstream images first |
| `docker compose down && docker compose up -d` | removes containers *and* the network before rebuilding. Bind mounts under `/srv/appdata` and `/mnt` are untouched — no data is at risk — but everything stops at once, including the tunnel. |

**If instead it created a network and then did nothing useful, check the project
name.** Compose derives it from the directory, and the GitHub repo is
`server_stack` while the checkout is supposed to be `~/stack` (§7). Cloned to the
wrong name, you get a *second* project — `server_stack_default` alongside
`stack_default` — that owns none of the running containers:

```bash
docker compose ls                                  # projects compose knows about
docker ps --format '{{.Names}}\t{{.Label "com.docker.compose.project"}}'
```

`COMPOSE_PROJECT_NAME=stack` in `.env` pins it regardless of the directory name,
and is set in `.env.example` for exactly this reason.

The symptom from the other side is a half-finished `up` that aborts on the first
name already taken:

```
✔ Network stack_default  Created
✘ Container jellyseerr   Conflict. The container name "/jellyseerr" is already in use
```

Every `container_name` here is fixed, so two projects cannot coexist — the second
one claims a name the first already holds. Clearing it is safe: **nothing in this
stack uses a named volume.** Every byte lives in a bind mount under `/srv/appdata`
or `/mnt`, so removing containers destroys no state at all.

```bash
cd ~/stack
docker ps -a --format 'table {{.Names}}\t{{.Label "com.docker.compose.project"}}\t{{.Status}}'
docker compose down --remove-orphans            # whatever this project half-created
docker rm -f jellyseerr                         # and any other name that conflicts
docker compose up -d
```

If the strays all carry an old project label, remove them in one go rather than
by name:

```bash
docker rm -f $(docker ps -aq --filter label=com.docker.compose.project=<old-name>)
```

Restart blast radius, worth internalising:

| Restarting | Also takes down |
| --- | --- |
| `gluetun` | qBittorrent, Prowlarr (namespace tenants) |
| `cloudflared` | every public hostname, ~10s |
| `qbittorrent`, `prowlarr` | only itself |
| `radarr`, `sonarr`, `bazarr`, an app | only itself |
| `mosquitto` | nothing, but Zigbee2MQTT reconnects and HA's entities go briefly unavailable |
| `homeassistant`, `zigbee2mqtt` | only itself |

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
| transcodes stay software-only despite `/dev/dri` | Plex Pass missing, or the Transcoder setting not ticked | [media/README.md](media/README.md) |
| `/dev/dri` present on the host, absent in the container | `docker compose` was given `-f`, or `COMPOSE_FILE` omits `hw/x86.yaml` | run bare `docker compose` from `~/stack` (§3) |
| `/` is ~100 GB on a 500 GB disk | Ubuntu's guided LVM left the VG unallocated | `lvextend -l +100%FREE` + `resize2fs` (§1) |
| a published port is reachable despite a ufw rule | Docker's rules sit ahead of ufw's chains | delete the `ports:` entry; ufw cannot close it (§2) |
| every container restarted overnight | `unattended-upgrades` updated docker-ce | `apt-mark hold` the docker packages (§2) |
| `conflicting options: port publishing and the container type network mode` | a `ports:` block on a service using `network_mode: service:gluetun` | move the port to gluetun's `ports:` |
| `port is already allocated` | host port collision | pick a free host port, update §4 |
| Radarr/Sonarr cannot reach qBittorrent or the indexers | namespace tenants have no DNS name | `http://gluetun:8080` and `http://gluetun:9696`, never their own names (§5) |
| every torrent errors the moment it is added | the default save path is still the image's `/downloads`, which this stack does not mount — the only media mount is `/data` | *Options → Downloads → Default Save Path* → `/data/incomplete` ([media/README.md](media/README.md)) |
| torrents error on a path that does exist | `/mnt/active` is not writable by the container user | `sudo chown -R 1000:1000 /mnt/active`; `docker compose exec qbittorrent touch /data/probe` proves it either way |
| one service is unreachable while its namespace-mate answers | that container was never created — an earlier `up` aborted partway on a name conflict | `docker compose ps -a`; §12 clears the conflict, then `up -d` again |
| Prowlarr's qBittorrent client fails against host `gluetun` | Prowlarr is *inside* that namespace, and gluetun's DNS proxy replaced Docker's resolver | host `localhost`, port 8080 (§5) |
| HA discovers nothing it used to, no error anywhere | Plex's DLNA holds UDP 1900, or `avahi-daemon` holds 5353 | [home/README.md](home/README.md) — disable Plex's DLNA server, and avahi if unused |
| Zigbee2MQTT cannot open the adapter | the ConBee is on a different port, or deCONZ is running and holds it | `ls -l /dev/serial/by-id/`; the two are mutually exclusive ([home/README.md](home/README.md)) |
| MQTT clients connect and are refused | mosquitto 2.x defaults to localhost-only, anonymous denied | `config/mosquitto.conf` was never written — [home/README.md](home/README.md) has the file |
| Prowlarr worked for weeks, then finds nothing | the TorrentDay cookie expired and the indexer was auto-disabled | redo [media/README.md](media/README.md) step 2; check *Indexers* for a disabled one |
| Prowlarr's indexer test returns a Cloudflare challenge | the cookie is bound to your browser's IP, not the VPN exit | FlareSolverr as an indexer proxy ([media/README.md](media/README.md)) |
| `up -d` creates a network and reports no containers | the checkout directory name differs from the running containers' compose project | `COMPOSE_PROJECT_NAME=stack` in `.env` (§12) |
| `Conflict. The container name "/gluetun" is already in use` | same cause, seen from the other side: a second project trying to claim fixed `container_name`s | as above; `docker compose down` under the old project first |
| Prowlarr cannot reach `radarr:7878` when adding the App | gluetun's killswitch is dropping outbound traffic to the docker network | add `FIREWALL_OUTBOUND_SUBNETS=<the compose network subnet>` to gluetun; `docker network inspect stack_default` names it |
| `qBittorrent login failed` | temporary password rotated on restart | permanent password + subnet whitelist ([media/README.md](media/README.md)) |
| upload is always 0 and the status bar shows *No direct connections* | NordVPN forwards no port, so nobody can connect in | structural — [media/README.md](media/README.md), "Why nothing uploads" |
| torrents stop by themselves shortly after finishing | a qBittorrent seeding limit, or Radarr removing completed downloads | [media/README.md](media/README.md); on a tracker paying for uptime, both are worth turning off |
| a download completes and Radarr never imports it | usually one of five things, and *Activity → Queue* names which | [media/README.md](media/README.md), "When a download completes and nothing gets imported" |
| a film grabbed from Prowlarr's Search tab never imports | Radarr only tracks downloads it started, in its own category | search from Radarr for anything Radarr should own ([media/README.md](media/README.md)) |
| imports copy instead of hardlinking, and the disk fills twice | `downloads/` and `library/` are on different filesystems, or *Use Hardlinks* is off | both live under `/mnt/active` (§1); confirm with `ls -l` showing link count 2 on an imported file |
| Bazarr's library is empty | Radarr/Sonarr not connected, or connected with the wrong port | `radarr:7878`, `sonarr:8989` — bridge names, not `gluetun` ([media/README.md](media/README.md)) |
| Jellyseerr requests approve but nothing downloads | *Settings → Services* has no Radarr/Sonarr, or the wrong root folder | [apps/README.md](apps/README.md) |
| an `.env` change appears to have no effect | `docker compose restart` reuses the container, and environment is fixed at creation | `docker compose up -d <service>` to recreate it |
| `cloudflared` restarts in a loop | it is exiting, not retrying — nearly always a bad or stale tunnel token | `docker compose logs cloudflared` names it (§9); the LAN is unaffected meanwhile |
| `502`/`530` from a public hostname | cloudflared cannot resolve or reach the service | service name and **container** port; is it in the same project? |
| Cloudflare hostname 404s all assets | a Path prefix was set | leave Path empty, use a subdomain |
| `docker compose exec qbittorrent curl ifconfig.me` shows the ISP IP | namespace sharing not in effect | check `network_mode`, recreate |
| Ubuntu stuck at boot with no network | a USB SSD did not mount | `nofail` in `/etc/fstab` (§1) |
| that same command hangs | gluetun killswitch, usually DNS | `docker compose logs gluetun` |
| Plex shows half-finished files | downloading straight into a library | set the incomplete path ([media/README.md](media/README.md)) |
| finished torrents take minutes to "move" | a category path escaped the `/data` mount | every category lives under `/data`, one disk (§1, [media/README.md](media/README.md)) |
| Plex cannot delete a file | `/data` and `/archive` are mounted read-only | intended; delete through Radarr or Sonarr ([media/README.md](media/README.md)) |
| moved a folder to archive and seeding stopped | qBittorrent only mounts `/mnt/active` | expected — the trade described in §1 |
| the active disk filled up with no warning | the finder's free-space guard is gone | *Minimum Free Space* in Media Management, plus `df` in your routine ([media/README.md](media/README.md)) |
| a container exits at start | a `${VAR:?...}` has no value | fill it in `~/stack/.env` |
| disk full, no obvious cause | old build layers | `docker image prune -f`, `docker system df` |

## 14. Security posture

- Nothing but Cloudflare is exposed: no router ports are forwarded, and the
  tunnel is outbound-only.
- Public apps get **two** locks: a Cloudflare Access policy and the app's own
  auth. Radarr, Sonarr, Bazarr and qBittorrent can all rewrite or delete the
  library, which is exactly why none of them has a hostname (§9) — the tunnel
  carries Jellyseerr and matte-vm only.
- **Home Assistant is the least contained thing on this box**, and knowingly so:
  `network_mode: host` plus `privileged: true` gives it the host's network stack
  and its devices. Host networking it genuinely needs, for discovery. `privileged`
  is worth questioning: it exists for direct hardware access, and the only radio
  here is the ConBee, which belongs to zigbee2mqtt. Try removing it — if nothing
  breaks, that is a real reduction in the blast radius of the one container that
  has none. Either way it gets no public hostname, its own login is the only lock
  that matters, and it needs patching for the same reason Jellyseerr does.
- **Be honest about what "no hostname" buys.** Jellyseerr is the one internet-
  facing app that holds Radarr's and Sonarr's API keys, and it sits on the same
  docker network as them. Not being tunnelled stops anyone reaching Radarr
  *directly*, but it does not contain a Jellyseerr compromise: code running in
  that container can reach `radarr:7878` with a valid key, and Radarr can move
  and delete files. Network isolation buys nothing here, because Jellyseerr has
  to reach them to do its job. Keeping Jellyseerr patched is the actual control,
  which is a reason to prefer a pinned tag you bump deliberately over discovering
  a version six months old (§10).
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
  site: NordVPN service credentials, the tunnel token, and any of the app API
  keys — Prowlarr's in particular, since it fronts the tracker account.
