# homelab runbook

Everything needed to rebuild this server from a blank disk, plus how to change
or extend it afterwards. The hardware is a BOSGAME mini PC (Intel N95) running
Ubuntu Server 24.04. The Raspberry Pi it replaces is still a supported target
on the `pi` branch; §15 covers restructuring the disks and moving between them.

**What runs here**

| Stack | Services | Reachable at |
| --- | --- | --- |
| media | gluetun (VPN), qBittorrent, Prowlarr, Plex | LAN ports |
| automation | Radarr, Sonarr, Bazarr | LAN ports only — admin tools |
| home | Home Assistant, Mosquitto, Zigbee2MQTT | LAN ports only |
| matte-vm | FastAPI + SQLite + PWA | its own Cloudflare hostname |
| jellyseerr | request front-end for Plex | its own Cloudflare hostname |
| kometa | Plex collections and posters | nothing — scheduled job, no UI |
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

/mnt/active/           1 TB USB — everything torrents touch
  incomplete/          qBittorrent's default save path AND its incomplete path
  downloads/movies/    qBittorrent's targets; torrents keep seeding from here
  downloads/tv/
  library/movies/      Radarr's and Sonarr's output; what Plex reads
  library/tv/
  books/               manual grabs — ebooks, audiobooks, magazines
  other/               manual grabs — anything else

/mnt/archive/          500 GB USB — content moved off active once it fills
  movies/  tv/  ...    read-only; mirrors whichever active folders you move
```

**Why `downloads/` and `library/` both exist — nothing is duplicated.** On Linux a
file is an *inode*: the data plus its metadata. A directory entry is just a
*name* pointing at an inode, and a **hardlink** is a second name for the same
one. There is no original and no copy — one set of blocks on disk, two ways to
reach it, and `df` counts it once.

That is what lets two consumers make incompatible demands on the same bytes:

- **qBittorrent** must keep the file exactly as the torrent describes it —
  original release name, original folder structure — or the torrent breaks and
  seeding stops.
- **Plex** wants `Some Film (2019)/Some Film (2019).mkv`, and wants none of the
  sample clips, `.nfo` files and nested folders a release ships with.

So one film is `downloads/movies/Some.Film.2019.1080p.WEB-DL.x264-GRP/` to the
tracker and `library/movies/Some Film (2019)/` to Plex, and it occupies its size
**once**.

Three directories, three stages of the same file:

| Directory | What is in it |
| --- | --- |
| `incomplete/` | partial data being written; not playable, not seeding yet |
| `downloads/` | complete, original names — what qBittorrent seeds from |
| `library/` | the same inodes under tidy names, organised by Radarr and Sonarr |

Every transition is free because all three are one filesystem: completing is a
rename from `incomplete/` into `downloads/`, and importing is a hardlink into
`library/`. Neither copies a byte.

**The two names resolve to one by themselves.** When a torrent has met its
seeding requirement and qBittorrent removes it, the `downloads/` name is deleted,
the inode's link count drops from two to one, and the file carries on living in
`library/` untouched. No copying, no cleanup by hand, no moment where the disk
holds it twice.

**Does the final location matter for seeding? Yes — which is exactly why Radarr
never moves the file.** qBittorrent seeds from the path it recorded; move the file
and it reports missing data and stops. Radarr's *Use Hardlinks instead of Copy*
(§8) is what leaves the original name in place. If it ever *cannot* hardlink —
because the two trees ended up on different filesystems — it silently falls back
to copying: seeding still works, but every film now genuinely occupies twice its
size. A library growing about twice as fast as it should is the symptom, and §13
has it.

**One caveat on not minding about seeding.** TorrentDay is a private tracker, and
private trackers generally enforce a ratio and hit-and-run rules, so dropping
every torrent the moment it finishes is how accounts get disabled — check its own
rules. Letting the old Pi-era torrents go (§15c) is harmless; they have had their
run. Never seeding at all is a different decision, and the arrangement above is
precisely what makes seeding cost nothing: the bytes Plex is serving are the same
bytes you are seeding.

**`books/` and `other/` are the manual half.** Radarr covers movies, Sonarr
covers TV, and nothing here covers an occasional ebook or audiobook — so those
are grabbed by hand from Prowlarr's own Search tab (§8) into their own
qBittorrent categories. Inside `books/`, keep `audiobooks/`, `ebooks/` and
`magazines/` as subfolders if you want them apart. Nothing on this box reads
them; that would be Calibre-web or Audiobookshelf, added per §10.

**Why state is internal and media is not.** Plex's library is a SQLite database
bound by random writes, so its location is the one you can actually feel; media
is sequential, re-downloadable, and far too big for 500 GB. All three disks are
SSDs, so the mountpoints name each disk's *job* rather than its bus.

**Why one disk holds everything torrents touch.** qBittorrent gets a single
mount — `/mnt/active` as `/data` — so `incomplete/`, `movies/`, `tv/` and
`other/` are unavoidably on one filesystem. Completing a torrent is then a
rename instead of a copy, and it cannot regress into a copy later by someone
pointing one category at the other disk — and the same one filesystem is what
lets Radarr and Sonarr import by hardlink instead of copying (§8).

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

Plex finds it again, because both directories are in the same library (§8).
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
  README.md                       this file
  compose.yaml                    include: list + cloudflared
  media/compose.yaml              gluetun, qbittorrent, prowlarr, plex
  media/arr.yaml                  radarr, sonarr, bazarr
  home/compose.yaml               homeassistant, mosquitto, zigbee2mqtt
  apps/matte-vm.yaml              one fragment per standalone app
  apps/jellyseerr.yaml            third-party image, same shape
  apps/kometa.yaml                scheduled job: no port, no hostname
  hw/x86.yaml                     the only per-hardware overlay, picked in .env
  bin/autodeploy                  poll git, redeploy what changed
  autodeploy.conf                 repo -> service map
  systemd/autodeploy.{service,timer}   installed into /etc (§11)
  .env                            ALL secrets, gitignored
  .env.example                    same keys, no values, committed
  .gitignore
~/git/matte-vm/                   app repo: Dockerfile + its own compose file
~/git/torrent_finder/             retired from the stack (§8), repo kept
~/git/<next-app>/                 same shape
/srv/appdata/<group>/<service>/   container state, internal SSD; <group> is the
                                  compose fragment's own directory (§1)
/mnt/active/, /mnt/archive/       bulk media, USB drives (§1)
```

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
| 8082 | Zigbee2MQTT frontend | deCONZ wants the same port — see §8 |
| 8123 | Home Assistant | `network_mode: host`, so no `ports:` entry |
| 32400 | Plex | `network_mode: host` |

Container ports never collide, only host ports. An app reachable solely through
the tunnel needs no `ports:` entry at all, and a batch job like `kometa` needs
none either — it never listens.

## 5. The rule that breaks things first

`qbittorrent` and `prowlarr` use `network_mode: "service:gluetun"`, so they have
**no network identity of their own**:

**The address depends on which side you are asking from**, and this is the part
that is easy to get wrong:

| From | To | Address |
| --- | --- | --- |
| a bridge service — Radarr, Sonarr, Bazarr, Jellyseerr | qBittorrent | `http://gluetun:8080` |
| a bridge service | Prowlarr | `http://gluetun:9696` |
| **Prowlarr** (inside the namespace) | **qBittorrent** | **`http://localhost:8080`** |
| anywhere | Radarr, Sonarr, Bazarr | their own names — ordinary bridge services |

- `http://qbittorrent:8080` and `http://prowlarr:9696` never resolve from
  anywhere. Neither name exists.
- **Prowlarr and qBittorrent are already on the same loopback**, so between those
  two the answer is `localhost` — not a workaround but the correct address, and
  the only one that needs no DNS at all.
- Using `gluetun` from *inside* the namespace does not work either, even though
  it looks symmetrical: gluetun replaces the container's resolver with its own
  DNS-over-TLS proxy, so Docker's service-name resolution at `127.0.0.11` is gone
  for its tenants. The name is looked up upstream and does not exist.
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
| `WIREGUARD_PRIVATE_KEY` | NordVPN | gluetun |
| `CLOUDFLARE_TUNNEL_TOKEN` | Cloudflare | cloudflared |
| qBittorrent's WebUI login | qBittorrent's API | Radarr, Sonarr and Prowlarr, each in its own UI |
| Prowlarr's / Radarr's / Sonarr's API keys | each other | entered in each app's UI, never in `.env` |

## 7. First-time setup, in order

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
sudo mkdir -p /mnt/active/{incomplete,downloads/{movies,tv},library/{movies,tv}} \
              /mnt/active/{books/{audiobooks,ebooks,magazines},other} \
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

**Logging in, before anything else.** These apps do not share credentials and
none of them ships a default pair, so there is nothing to look up — you create
each one, once:

| App | Where the first login comes from |
| --- | --- |
| Prowlarr, Radarr, Sonarr | **you invent it** on first visit. Each shows an *Authentication Required* setup screen before anything else and will not proceed until you pick a method and set a username and password. |
| Bazarr | *Settings → General → Security*, off by default; set it there. |
| qBittorrent | the **only** one with a generated password — `admin` plus a temporary one printed in the log on every start until you set a permanent one (below). |
| Plex, Jellyseerr | your Plex account, via plex.tv. |

That third row is the one that causes the confusion: qBittorrent's
password-in-the-logs behaviour is specific to it, and looking for the equivalent
in Prowlarr's logs will not find one.

Four separate logins for four admin tools is tedious, and they are all LAN-only
by design (§9), so *Authentication Required → Disabled for Local Addresses* is a
defensible setting on the three \*arr apps. Understand what it means first:
they treat any RFC1918 source as local, and behind Docker every LAN request
arrives from a private address, so in practice it means **anyone on your network
has full control** of your library. The containment is that the LAN is the
boundary — nothing here is on the tunnel.

**Lost one?** The \*arr apps keep their settings in `config.xml` next to their
database, so recovery is a file edit rather than a reinstall:

```bash
cd ~/stack && docker compose stop prowlarr        # or radarr / sonarr
sudo sed -i 's|<AuthenticationMethod>.*</AuthenticationMethod>|<AuthenticationMethod>None</AuthenticationMethod>|' \
  /srv/appdata/media/prowlarr/config.xml
docker compose start prowlarr                     # then set a new one in the UI
```

The same file holds the API key, which saves a click when wiring the apps to each
other:

```bash
grep -o '<ApiKey>[^<]*' /srv/appdata/media/prowlarr/config.xml
```

### qBittorrent (`http://<host>:8080`)

linuxserver's image prints a **new temporary password on every start** until a
permanent one is set, which would break Radarr and Sonarr on each reboot.

```bash
docker compose logs qbittorrent | grep -i "temporary password"
```

1. Log in, then *Options → Web UI → Authentication*: set a real password and put
   it in `QBITTORRENT_PASSWORD`.
2. Same page: tick **Bypass authentication for clients in whitelisted IP
   subnets** → `172.16.0.0/12`. Docker bridge networks live in that range, so
   Radarr, Sonarr and Prowlarr authenticate regardless of password changes. The
   WebUI is inside gluetun's namespace and not internet-reachable either way.
3. *Options → Downloads*: **do this before adding anything**, or every torrent
   errors immediately. The image ships a default save path of `/downloads`, and
   this stack does not mount that — the only media mount is `/data`, so the
   default points at nothing.
   - **Default Save Path**: `/data/incomplete`
   - **Keep incomplete torrents in**: `/data/incomplete`
4. *Categories*. Radarr and Sonarr create and own theirs on first contact — do
   not make them by hand. You only create the two manual ones (right-click the
   sidebar → Add category):

   | Category | Save path | Owned by |
   | --- | --- | --- |
   | `radarr` | `/data/downloads/movies` | Radarr, created automatically |
   | `sonarr` | `/data/downloads/tv` | Sonarr, created automatically |
   | `books` | `/data/books` | you, for manual Prowlarr grabs |
   | `other` | `/data/other` | you, for manual Prowlarr grabs |

   Note that only the manual categories save into their final home. Radarr's and
   Sonarr's land in `downloads/`, and the file reaches `library/` when the *arr
   app imports it — which is the whole point of the split (§1).

Every path above is under the single `/data` mount, which is the whole active
disk (§1). That is what makes completion a rename and an import a hardlink, and
it is why `incomplete/` is a sibling rather than living inside a library folder:
incomplete files inside a Plex library get scanned half-written and pollute it.

**`/data` must be spelled identically everywhere.** qBittorrent, Radarr, Sonarr
and Bazarr all mount `/mnt/active` at `/data` for one reason: qBittorrent reports
a finished torrent's location as a path, and Radarr has to open that exact path
to import it. Mount it as `/downloads` in one and `/data` in another and imports
simply never happen, with no error that names the cause (§13).

**Lost the password entirely:** stop the container first (qBittorrent rewrites
its config on exit), then either clear the hash to get a fresh temporary
password, or write a known one:

```bash
cd ~/stack && docker compose stop qbittorrent
cp /srv/appdata/media/qbittorrent/qBittorrent/qBittorrent.conf{,.bak}
sed -i '/^WebUI\\Password_PBKDF2=/d' /srv/appdata/media/qbittorrent/qBittorrent/qBittorrent.conf
docker compose up -d qbittorrent
docker compose logs qbittorrent | grep -i "temporary password"
# or: python3 ~/git/torrent_finder/deploy/raspberry-pi/qbt_password.py
#     and paste the line under [Preferences]
```

#### Why nothing uploads

Almost certainly not a qBittorrent setting: **NordVPN does not do port
forwarding.** Seeding effectively requires other peers to be able to open a
connection *to* you, and every route inbound is closed here:

- qBittorrent lives in gluetun's network namespace, so it announces the VPN exit
  node's address to the tracker. Peers dutifully try to reach you there, and
  NordVPN drops it — there is no port on their side mapped back to you.
- The `6881` published on gluetun in `media/compose.yaml` does **not** change
  that, and this is the misconception worth killing: publishing a port makes it
  reachable from your LAN, not from the internet. Inbound WAN traffic never
  arrives at the host at all, because the tracker never told anyone your home
  address.
- Forwarding 6881 on the router would not help either, for the same reason — and
  §14 keeps router ports closed on purpose.

So you are permanently "firewalled" in BitTorrent's terms. qBittorrent says so
itself: the status bar shows a red network icon and *No direct connections*.
Outbound-only, you can still upload to leechers you happen to connect to, which
on a private tracker's small, mostly-seeded swarms rounds to nothing. That
matches the symptom exactly.

**The NordVPN subscription is not wasted, and its P2P servers are not the answer
either.** Those servers permit and route torrent traffic well, which is a
different question from whether anyone can reach you; NordVPN offers no port
forwarding on any plan or server type. What the subscription still does, and does
fine:

- keeps the traffic off your ISP's view of you
- gives gluetun a killswitch, so nothing leaks when the tunnel drops
- downloads at full speed, because *outbound* connections to seeders are all a
  download needs

Only seeding is missing, and the bonus system is already covering the ratio. So
there is a legitimate do-nothing option here.

**A dedicated IP does not fix this, and might make it worse.** Nord's dedicated
IP add-on gives you an address nobody else shares, which solves CAPTCHAs,
shared-IP blocklists and services that want to whitelist you. It does not include
port forwarding — there is still nothing on Nord's side mapping an inbound port
back down your tunnel, which is the only thing that would make you connectable.
Worse, it is a distinct server type from the P2P-optimised ones, so check whether
P2P traffic is even permitted on it before paying; buying it for this could cost
you the working downloads you have.

**If you want seeding, it is a provider change.** What matters is whether they
forward a port and whether gluetun can ask for it automatically:

| Provider | Port forwarding | gluetun handling |
| --- | --- | --- |
| NordVPN *(current)* | none, on any plan | — |
| Private Internet Access | yes, all plans | native; gluetun negotiates the port and opens the firewall itself |
| ProtonVPN | yes, on paid plans | native, same as PIA |
| AirVPN | yes, and **static** — you pick it in their panel | no automation needed; set the same port in qBittorrent and in `FIREWALL_VPN_INPUT_PORTS` |
| Mullvad | removed in 2023 | — |

The rotating-port providers need the port pushed into qBittorrent on every
reconnect; AirVPN's static port avoids that entirely, which is worth something
against PIA being the cheapest. Either way, do it at Nord's renewal rather than
paying two subscriptions to fix something the bonus points are already absorbing.

The shape for a rotating port, with gluetun's own documentation as the authority
on exact variable names — they have moved between versions:

```yaml
      - VPN_SERVICE_PROVIDER=protonvpn
      - VPN_TYPE=wireguard
      - VPN_PORT_FORWARDING=on
      # The assigned port changes on reconnect, so push it into qBittorrent
      # rather than hardcoding one. Enable "Bypass authentication for clients on
      # localhost" in qBittorrent first, or this call gets a 403.
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -qO- --post-data="json={\"listen_port\":{{PORTS}}}" http://127.0.0.1:8080/api/v2/app/setPreferences'
```

Until then the bonus-point system is doing the work, and that is a perfectly
reasonable place to sit — but it depends on torrents staying **active**, which
brings us to the settings that can still stop you seeding even after inbound
works.

#### Settings that silently prevent seeding

| Setting | Trap |
| --- | --- |
| *Options → Speed → Global upload rate* | **`0` means unlimited, not zero.** To seed at a limited speed put a real number in, e.g. `2000` KiB/s. Leaving it at 0 is not what is stopping you. |
| *Options → BitTorrent → Seeding Limits* | if "when ratio reaches" or "when seeding time reaches" is set to *Stop torrent*, torrents park themselves and the bonus clock stops |
| *Options → Connection → Use UPnP/NAT-PMP* | turn it **off**; it cannot work through the VPN namespace and only adds log noise |
| *Options → BitTorrent → Anonymous mode* | off — it suppresses information private trackers expect |
| Radarr/Sonarr → *Download Clients → Remove Completed* | **the one that matters for you.** If Radarr deletes the torrent as soon as it has imported, the torrent stops being active and the bonus accrual with it. Set real seeding goals in qBittorrent and let Radarr clean up only after they are met. |

That last row is worth dwelling on, because it interacts with the hardlinks (§1).
Keeping a torrent seeding keeps the `downloads/` name alive alongside the
`library/` one — which costs nothing, since they are the same inode. There is no
disk-space reason to remove torrents early, and on a tracker that pays you for
uptime there is a reason not to.

### Prowlarr (`http://<host>:9696`)

Prowlarr is the indexer proxy: it holds the TorrentDay credentials once, and
pushes the indexer definition out to Radarr and Sonarr so they never hold it.

Steps 1–3 stand alone and are worth doing first. Steps 4–5 talk to other
containers, so they need Radarr and Sonarr already running with their API keys,
and qBittorrent already carrying a permanent password (§8) — come back for them.

**1. Note the API key.** *Settings → General → Security*, or straight out of
`config.xml` (§8 opening). Radarr, Sonarr and Jellyseerr each want it, entered in
their own UIs.

**2. Add TorrentDay.** *Indexers → Add Indexer → TorrentDay*. It authenticates
with a browser cookie rather than a password, and the reliable way to get one is
from a request rather than the cookie jar:

1. Log into TorrentDay in a browser, and **do not log out afterwards** — that
   invalidates the cookie you are about to copy.
2. F12 → *Network* → reload the page → click any request to the site →
   *Request Headers* → copy the entire value of the `Cookie:` header.
3. Paste the whole string into Prowlarr's **Cookie** field. It looks like
   `uid=...; pass=...; ...` and every part matters.

Copying individual cookies out of the *Application*/*Storage* tab works too but
is easy to get wrong; the request header is already formatted the way Prowlarr
wants it. Hit **Test** before saving.

**3. Search once from Prowlarr itself.** The *Search* tab, any common title.
This separates "the indexer works" from "the wiring to Radarr works", and you
want to know which you are debugging later.

**4. Point Prowlarr at Radarr and Sonarr.** *Settings → Apps → Add → Radarr*,
then again for *Sonarr*. Two addresses, and they are not symmetric:

| Field | Value | Why |
| --- | --- | --- |
| Radarr / Sonarr Server | `http://radarr:7878`, `http://sonarr:8989` | ordinary bridge services, so their own names resolve |
| Prowlarr Server | `http://gluetun:9696` | Prowlarr has no network identity of its own — §5 |
| API Key | from that app's *Settings → General* | |

Getting the second one wrong is the usual mistake: it is the address *Radarr*
will use to call back to Prowlarr, so it must be gluetun's. Indexers then sync
outward automatically, and Radarr's *Settings → Indexers* fills in by itself.

**5. Add qBittorrent as a download client.** *Settings → Download Clients → Add
→ qBittorrent*, host **`localhost`**, port `8080`, the WebUI login from §8.

Not `gluetun` — Prowlarr and qBittorrent share a network namespace, so they are
already on the same loopback, and `gluetun` does not resolve from inside it
(§5). Radarr and Sonarr *do* use `gluetun` for the same client, because they are
ordinary bridge containers reaching in from outside. Same download client, two
different addresses, depending on which side is asking.

One consequence of using loopback: the subnet whitelist from §8 covers
`172.16.0.0/12`, and `127.0.0.1` is not in it, so Prowlarr needs the real
username and password. Alternatively tick *Bypass authentication for clients on
localhost* in qBittorrent, which then also covers anything else inside the
namespace.

This is what lets the *Search* tab actually grab something, which is how the
occasional ebook or audiobook gets downloaded now — set the category to `books`
or `other` on the grab.

That last step is the whole manual workflow. Prowlarr's search is not as pleasant
as a purpose-built UI, but it is already here, already authenticated, and already
inside the VPN namespace, and the alternative was maintaining an app for
something that happens a few times a year.

**Prove the chain before trusting it.** Search anything small in Prowlarr, grab
it with category `other`, and follow it through — each check catches a different
break:

```bash
cd ~/stack

# 1. qBittorrent took the torrent, and put it where the category says
docker compose exec qbittorrent ls -la /data/incomplete /data/other

# 2. it is downloading through the VPN, not the ISP
curl -s ifconfig.me; echo                          # the box's own address
docker compose exec qbittorrent curl -s ifconfig.me # must differ, and be Swedish

# 3. it landed on the host where you expect
ls -la /mnt/active/other/
```

If step 1 shows the file under `/data/other` but step 3 shows nothing in
`/mnt/active/other`, the `/data` mount is not what you think it is (§8). If step
2 returns the same address twice, stop and fix the VPN before downloading
anything else — that is the one failure worth catching immediately.

**Two things that will fail later, not now.** The cookie expires — silently, weeks
in, and Prowlarr disables an indexer after enough consecutive failures, so
"nothing finds anything any more" usually means repeating step 2. And if TorrentDay
puts Cloudflare in front of itself, a cookie minted in your browser is bound to
your browser's address and will not work from the VPN exit; the standard answer is
a FlareSolverr container, added per §10 and pointed at from *Settings → Indexers →
Indexer Proxies*.

### Radarr (`http://<host>:7878`) and Sonarr (`http://<host>:8989`)

Identical setup; do both. Neither is on the tunnel — they can rewrite your
library, so they stay on the LAN (§9).

1. *Settings → Media Management → Root Folders → Add*:
   `/data/library/movies` for Radarr, `/data/library/tv` for Sonarr.
2. *Settings → Media Management*: turn on **Rename Movies** / **Rename
   Episodes**, and leave **Use Hardlinks instead of Copy** on. If it ever copies
   instead, `/data` is two filesystems and §1 has been violated.
3. *Settings → Download Clients → Add → qBittorrent*: host `gluetun`, port
   `8080`, the WebUI login from §8. Leave *Category* at `radarr` / `sonarr` — it
   creates them in qBittorrent for you.
4. *Settings → Indexers*: already populated by Prowlarr (§8). If empty, step 4
   of Prowlarr did not take.
5. *Settings → Profiles → Quality Profiles*: this is where the buffering fix
   from §8's release guidance stops being a habit. Build a profile that allows
   1080p WEB-DL and Bluray, rejects Remux and 2160p, and it will never again
   grab a file that forces a burn-in transcode.
6. *Settings → Profiles → Quality Definitions*: set the maximum sizes. This
   replaces the `MAX_TORRENT_SIZE_GB` guard the finder used to enforce.

**One guard did not survive the move.** The finder refused a download when free
space fell below `MIN_FREE_SPACE_GB`, measured on the download disk. Radarr and
Sonarr have a *Minimum Free Space* under *Media Management* (advanced), but it
defaults to a value far below 20 GB and only blocks the import, not the download.
Set it, and keep `df -h /mnt/active` in your routine (§12) — automation fills a
disk considerably faster than choosing releases by hand did.

### Bazarr (`http://<host>:6767`)

Bazarr has no library of its own: it asks Radarr and Sonarr what exists and
where, which is why it could not run here until now. Set it up after both of
those have imported your library, or it will connect to two empty catalogues and
look broken.

**1. Connect Radarr and Sonarr.** *Settings → Radarr* — address `radarr`, port
`7878`, Radarr's API key; then *Settings → Sonarr* with `sonarr` and `8989`.
These are ordinary bridge services, so their own names resolve; no `gluetun`
here (§5).

**Leave the path mapping table empty.** It exists because Bazarr is usually told
a path by Radarr that means something different inside Bazarr's own container,
and filling it in wrongly is the classic way to break this. It is unnecessary
here for a specific reason: qBittorrent, Radarr, Sonarr and Bazarr all mount
`/mnt/active` at the same container path, `/data` (§1), so a path Radarr reports
is a path Bazarr can open, unchanged. If Bazarr reports files it cannot find,
that mount agreement has broken — do not paper over it with a mapping.

**2. Build a language profile.** Three steps that people conflate, and nothing
downloads until all three are done:

1. *Settings → Languages → Languages Filter*: enable the languages you care
   about. This only makes them selectable.
2. *Languages Profiles → Add*: an ordered list — Swedish first, English second —
   with a **cutoff** at the point where Bazarr should stop looking. Decide here
   whether to accept *hearing impaired* and *forced* variants; HI subtitles are
   often the only ones available and are visibly noisier to read.
3. *Default Settings*: assign that profile as the default for Series and for
   Movies, separately.

**3. Apply the profile to what already exists.** This is the step that gets
missed. The default only applies to *newly added* items, so a library imported
before Bazarr existed has no profile and Bazarr will sit there doing nothing:

- *Series* → select all → **Mass Edit** → assign the profile.
- *Movies* → the same.

**4. Providers.** *Settings → Providers*. OpenSubtitles.com needs a free account
and has a daily download quota — anonymous use is limited enough to look like a
malfunction. Add a second provider such as Podnapisi so one quota does not stall
everything. The list changes as providers die, so treat whatever Bazarr offers
today as the menu.

**5. Subtitle behaviour.** *Settings → Subtitles*, four settings worth setting
deliberately:

| Setting | Why |
| --- | --- |
| **Use embedded subtitles** — on | do not fetch what the file already carries as text |
| **Minimum score** | lower finds more and syncs worse. Start at the default and lower it only for a language that keeps coming up empty |
| **Automatic subtitle synchronization** | fixes the offset that makes a technically-correct subtitle useless |
| **Upgrade previously downloaded subtitles** | lets a poor early match be replaced later, which matters when a release lands before its subtitles do |

Leave subtitles saved **alongside the video**, not embedded. That is the whole
point for this box: a sidecar `.srt` is text, so Plex hands it to the client and
nothing transcodes, where an embedded PGS forces burn-in (§8).

Bazarr writes those `.srt` files into `/data/library/...`, so it holds `/data`
read-write for the same reason Radarr does.

**Then turn Plex's own subtitle agent back off.** Two things fetching subtitles
for the same file produce duplicates that are tedious to unpick, and Bazarr is
much better at it — more providers, scoring, per-language rules, and upgrades
over time.

### Plex (`http://<host>:32400/web`)

First run only: uncomment `PLEX_CLAIM`, put a fresh token from
https://plex.tv/claim in `.env` (it expires in 4 minutes), start Plex, then
remove it again. Two libraries, each spanning both disks:

| Library | Type | Folders |
| --- | --- | --- |
| Movies | Movies | `/data/library/movies`, `/archive/movies` |
| TV Shows | TV Shows | `/data/library/tv`, `/archive/tv` |

Point Plex at `library/`, never at `downloads/`. Files in `downloads/` are
mid-import, wrongly named, and often duplicates of what is already in the
library — exactly the pollution the incomplete directory was kept out of.

Both mounts are read-only, so Plex's own *delete media* does nothing. Deleting
is Radarr's and Sonarr's job, and they hold `/data` read-write.

`books/`, `other/` and `downloads/` are deliberately outside every library. Plex
dropped ebook support years ago, so reading what lands in `books/` needs a
different app — Calibre-web or Kavita for ebooks and magazines, Audiobookshelf
for audiobooks. Adding one is §10, and it would mount `/mnt/active/books`
read-only; nothing about the layout has to change first.

**What to prefer when picking a release.** Hardware transcoding removes most of
the reason to care about format. Two things it does not fix:

- **Subtitles cost more than the codec does.** Burning in a subtitle forces a
  transcode by definition — direct play is impossible once a frame has to be
  redrawn. Text subtitles (SRT, ASS) are sent to the client and drawn there for
  free; image subtitles (PGS from Blu-ray, VOBSUB from DVD) are the ones almost
  no client can draw, so Plex burns them in. That is the whole difference
  between a 1080p WEB-DL with an SRT, which direct plays, and the same film as a
  Blu-ray remux with PGS, which transcodes every single time.

  A true *hardsub* release — subtitles already painted into the picture, common
  in anime rips — is not the same thing and costs nothing, because there is
  nothing left for Plex to overlay.

- **4K HDR is the case this box can still lose.** A 4K file the client direct
  plays is free. A 4K HDR file that has to be transcoded also needs HDR→SDR tone
  mapping, which is the most expensive thing Quick Sync does here and looks
  washed out even when it keeps up. Prefer 1080p unless the client that matters
  can direct play 4K HEVC.

Beyond those, don't constrain anything: H.264 vs HEVC, MKV vs MP4, DTS vs AC3
are all absorbed at 1080p without the N95 noticing, and `MAX_TORRENT_SIZE_GB=25`
already keeps 4K remuxes and most 1080p ones out — which filters most PGS
sources as a side effect.

Three settings make it stick:

| Where | Setting |
| --- | --- |
| Server → Transcoder | hardware acceleration **and** hardware-accelerated encoding on |
| Each client → Playback | *Burn subtitles* → **Only image formats**, never *Always* |
| Server → Agents / a subtitle agent | let Plex fetch SRTs, so an embedded PGS is never the only option |

When something buffers, the dashboard says why rather than leaving you to guess:
hover the active stream and it names Direct Play, Direct Stream or Transcode,
marks an offloaded transcode `(hw)`, and gives the reason — `subtitle burn-in`
and `container not supported` being the two you will actually see. Software
transcoding of 1080p on this box is a misconfiguration, not a capacity limit.

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

### Jellyseerr (`http://<host>:5055`, `requests.bjorngreen.se`)

Create the config directory first, or the container starts and cannot write:

```bash
sudo mkdir -p /srv/appdata/apps/jellyseerr && sudo chown -R 1000:1000 /srv/appdata/apps/jellyseerr
cd ~/stack && docker compose up -d jellyseerr
docker compose logs -f jellyseerr        # a permissions complaint here means the chown
```

**1. The wizard.** Choose **Plex**, and for the server address use
`host.docker.internal` port `32400` — not `plex`, which does not resolve (§10).
Sign in with your Plex account; the account that completes the wizard becomes the
Jellyseerr owner.

**2. Enable the libraries and scan.** *Settings → Plex → Libraries*: tick Movies
and TV Shows, then **Start Scan**. Skipping this is why availability detection
appears not to work — Jellyseerr only knows a request is fulfilled if it is
watching the library the file landed in. Set *Settings → General → Application
URL* to `https://requests.bjorngreen.se` at the same time, or Plex's OAuth
redirect and every notification link points at the wrong place.

**3. Wire the fulfilment**, which is what makes it more than an inbox —
*Settings → Services → Add Radarr Server*, and again for Sonarr:

| Field | Value |
| --- | --- |
| Hostname | `radarr` / `sonarr` — bridge services, so the names resolve (§5) |
| Port | `7878` / `8989` |
| API key | from that app's *Settings → General* |
| Quality profile | the 1080p profile from §8, so requests inherit the format policy |
| Root folder | `/data/library/movies` / `/data/library/tv` |
| **Default Server** | **tick it.** Nothing routes without it, and the failure is silent: requests approve and then sit there |

Leave the 4K server fields alone. They are for a second Radarr managing a
separate 4K library, which §8's release policy deliberately does not have.

**4. Decide who can ask for what.** *Settings → Users → Import Plex Users* pulls
in everyone who has access to your Plex server; they can also just sign in once
and appear. Then per user, or as the global default under *Settings → Users*:

| Permission | What it means here |
| --- | --- |
| Request | can ask. The baseline. |
| **Auto-Approve** | their requests go straight to Radarr with no queue. Right for you and for people whose taste you do not need to review |
| Request quota | requests per week or day. The only real guard against one person filling `/mnt/active` |
| Manage Requests | can approve other people's — keep it to yourself |

The quota is worth setting even for trusted users, because the free-space guard
that torrent-finder used to enforce is gone (§8) and automation fills a disk
quickly.

**5. Get told when something is requested.** *Settings → Notifications* — Discord
or email is enough. Without it, a request that needs approval waits for you to
happen to open the site, which rather defeats the point of the inbox.

A request now goes: someone asks → you approve (or auto-approve trusted users) →
Radarr searches Prowlarr → qBittorrent downloads through the VPN → Radarr imports
and renames into `library/` → Plex scans → Jellyseerr marks it *Available*. No
step in that chain is yours once the profile is right.

**6. The intended flow: the Plex watchlist.** Plex has no extension mechanism, so
there is no request button in the Plex apps (§10). What there is, is Plex's own
**Add to Watchlist** button on anything in *Discover* — and Jellyseerr can treat
that as the request:

1. *Settings → Plex → Watchlist Sync* (or the equivalent under Plex settings):
   enable it, and let it run once so the first poll lands.
2. *Settings → Users → <user> → Edit*: enable **Auto-Request** for movies and
   series, and set the quality profile and root folder those requests inherit.

Note this is **polling, not a webhook.** Jellyseerr asks the Plex API what is on
each user's watchlist on a timer; nothing is pushed to it, and there is nothing to
configure on the Plex side. Jellyseerr's own webhooks exist but point the other
way — outbound notifications to Discord and the like.

The one prerequisite that catches people: **each user must sign into Jellyseerr
once.** Watchlists are per-Plex-account and reading one needs that account's
token, which Jellyseerr only gets when the user completes the Plex sign-in. After
that single visit they never need to return — they add to their watchlist in the
Plex app and it appears in Radarr.

**The site is the fallback**, for anything the watchlist route handles badly:

| | Watchlist | Jellyseerr's own UI |
| --- | --- | --- |
| Where | inside the Plex app, no second site | `requests.bjorngreen.se`, signed in with their Plex account |
| Latency | polled, so minutes not seconds | immediate |
| TV granularity | the whole series | pick seasons |
| Control | takes your defaults | quality profile, and the approval queue |

So: watchlist for everyone as the default path, and the site for yourself, for
picking seasons, and for anyone whose taste needs an approval queue in front of
it. Either way nobody gets a new password — Jellyseerr signs people in with Plex,
so access to your server *is* the account.

**Keeping the public side safe.** Plex sign-in is real authentication, not a
token in a URL: it is OAuth against plex.tv, and Jellyseerr then checks the
account against your server's user list, so a stranger with a valid Plex account
still gets refused. Four things make that hold up on a public hostname:

| Do | Why |
| --- | --- |
| *Settings → Users*: turn **local sign-in** off | leaves Plex OAuth as the only path and removes password brute-forcing as a category entirely |
| Cloudflare → Security → **rate limiting** on the hostname, plus Bot Fight Mode | keeps the surface public for real users while making scanning and stuffing pointless; both are on the free plan |
| Cloudflare → WAF: geo-restrict to the countries your users are actually in | cheap, and cuts most background noise |
| Enable **2FA on your Plex account** | this is the highest-leverage item on the list: your Plex account is now the admin credential for the request system |

Cloudflare Access on top is the strongest option and stays the right answer if
you are the only requester — see §9 for why it stops being right the moment
anyone else is.

### Kometa (no UI — `docker compose logs kometa`)

Kometa refuses to start without a config, so write one before first run:

```bash
sudo mkdir -p /srv/appdata/apps/kometa && sudo chown -R 1000:1000 /srv/appdata/apps/kometa
# the image ships a fully commented template; take it and edit
docker run --rm kometateam/kometa:latest cat /config/config.yml.template \
  | sudo tee /srv/appdata/apps/kometa/config.yml >/dev/null
sudo nano /srv/appdata/apps/kometa/config.yml
```

Three values make it work, and the Plex one is the only awkward part:

| Setting | Value |
| --- | --- |
| `plex.url` | `http://host.docker.internal:32400` — **not** `http://plex:32400` (§10) |
| `plex.token` | in Plex web, open any item → *Get Info* → *View XML*; the `X-Plex-Token` in that URL |
| `tmdb.apikey` | free from themoviedb.org → Settings → API |

Then a first run you actually watch, rather than discovering the result at 05:00:

```bash
cd ~/stack && docker compose up -d kometa
docker compose logs -f kometa
```

**Start with collections, add overlays later.** Kometa applies overlays — the
4K/HDR badges on posters — by uploading modified artwork into Plex, so they are
not a view but a change to your library, and backing them out is fiddlier than
adding them. Collections and sort titles are cheap to undo by comparison. The
real escape hatch if a run makes a mess is the Plex database from the §12 backup,
which is heavy but genuine, so take one before the first overlay run.

Nothing here is on a hostname or a port. If Kometa appears to do nothing, it is
almost always the schedule: `KOMETA_TIME` and the container's timezone decide
when it wakes, and the logs say what it did.

### Home Assistant, Mosquitto and Zigbee2MQTT

`http://<host>:8123`, `http://<host>:8082` for the Zigbee frontend. Mosquitto has
no UI. All three keep their existing configuration — this was a working setup
moved in, not a new install, so §15d is the whole job.

**Two conflicts with what was already running.** Neither is a port clash; the
port registry (§4) is clean. Both come from Home Assistant needing the host's own
network interface for discovery:

| Conflict | Why | Fix |
| --- | --- | --- |
| **UDP 1900 — Plex's DLNA vs HA's SSDP** | both are host-networked and both want the SSDP port. Plex binds it for its DLNA server; HA binds it to discover UPnP devices | turn Plex's DLNA off: *Settings → Server → DLNA → uncheck **Enable the DLNA server***. You almost certainly do not use it — it exists for smart TVs that cannot run a Plex app |
| **UDP 5353 — HA's zeroconf vs `avahi-daemon`** | Raspberry Pi OS ships avahi enabled, and it holds mDNS. HA's zeroconf then finds nothing and silently discovers no Chromecasts, printers or ESPHome nodes | `sudo systemctl disable --now avahi-daemon` if nothing else needs it, or accept that HA discovery is manual |

Symptoms are quiet in both cases: nothing errors, discovery just comes up empty.
If HA finds no devices it used to find, start here.

**Zigbee2MQTT and deCONZ are mutually exclusive**, twice over: one ConBee II
cannot be claimed by two containers, and both want host port 8082. The compose
file keeps deCONZ as a comment for that reason.

**The ConBee's device path is already right.** `/dev/serial/by-id/usb-dresden_…-if00`
encodes the stick's serial, so it survives reboots, a different USB port, and the
move to the mini PC — where `/dev/ttyACM0` would not. Verify after any hardware
change:

```bash
ls -l /dev/serial/by-id/
docker compose exec zigbee2mqtt ls -l /dev/ttyACM0
```

**Mosquitto will not start usefully without its config.** Version 2.x listens on
localhost only and refuses anonymous clients until told otherwise, so
`/srv/appdata/home/mosquitto/config/mosquitto.conf` is required rather than optional.
Carrying the existing one over is the first thing §15d does; a broker that
accepts no connections looks exactly like a broker that is down.

**Who talks to whom**, since the three sit on different networks:

| From | To | Address | Why |
| --- | --- | --- | --- |
| Zigbee2MQTT | Mosquitto | `mqtt://mosquitto:1883` | both ordinary bridge services |
| Home Assistant | Mosquitto | `localhost:1883` | HA is host-networked, so it uses the *published* port, not the service name |

That second row is the one that looks wrong and is not: a host-networked
container is not on the compose network, so `mosquitto` does not resolve from it
— the same rule as §5, from the same direction as Jellyseerr reaching Plex (§10).

**Not on the tunnel, deliberately.** Home Assistant controls your house and runs
privileged; §14 covers why it gets no hostname. If you do want it remotely, it is
§9 plus an Access policy, and worth more thought than the media apps needed.

### Retired: torrent-finder

Removed from the stack. It was a manual search-and-grab front-end, and once
Radarr and Sonarr own release selection through quality profiles the only job
left was the occasional ebook — which Prowlarr's own *Search* tab does (§8),
without an internet-facing login to maintain for something that happens a few
times a year.

The repo is untouched at `~/git/torrent_finder` and still runs standalone
(`docker compose up` in it), so nothing is lost if this turns out to be wrong.
What went away with it, deliberately:

| Gone | Replaced by |
| --- | --- |
| picking releases by reading quality badges | Radarr/Sonarr quality profiles (§8) |
| `MAX_TORRENT_SIZE_GB` | Quality Definitions maximum sizes |
| `MIN_FREE_SPACE_GB` | *Minimum Free Space* plus `df` — a real downgrade, see §8 |
| `movies`/`tv`/`music`/`books`/`other` category routing | `radarr`/`sonarr` automatic, `books`/`other` manual |
| a request-shaped UI reachable from anywhere | Jellyseerr, which is what it should have been |

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

```bash
curl -sI https://requests.bjorngreen.se | head -1
# 401 = the app answered and refused for lack of credentials (healthy)
# 502/530 = cloudflared could not reach the service — name or port wrong
# Cloudflare login page = Access is in front (expected once §9 is done)
```

## 10. Adding a new app

### One you build from a repo

The app repo needs nothing but a `Dockerfile`; the stack owns how it is wired in.

1. Clone it into `~/git/<new-app>`.
2. Write `~/stack/apps/<new-app>.yaml` from the skeleton below — one service,
   `container_name` set so the tunnel URL is predictable, state under
   `/srv/appdata/apps/<new-app>`, and a host port from §4 only if you want LAN
   access.
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
      - /srv/appdata/apps/<new-app>:/data   # internal SSD, outside the checkout
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

### One that ships as an image

Third-party apps — Jellyseerr, a reverse proxy, an ebook server — differ in three
ways: `image:` instead of `build:`, **no** `autodeploy.conf` line (there is no
repo to poll, so they update only when you `docker compose pull`), and a tag
worth pinning, because anything with a database migrates it forward on start and
a migration does not roll back when you downgrade.

```yaml
# ~/stack/apps/<new-app>.yaml
services:
  <new-app>:
    image: vendor/<new-app>:1.2.3     # pin; `latest` is a silent major upgrade
    container_name: <new-app>
    environment:
      TZ: "${TZ:-Europe/Stockholm}"
    volumes:
      - /srv/appdata/apps/<new-app>:/config
    ports:
      - "50xx:50xx"
    restart: unless-stopped
```

**If it needs to talk to Plex, it cannot use the service name.** Plex runs with
`network_mode: host`, so it is not on this project's network and
`http://plex:32400` does not resolve — the same shape of trap as §5, from the
other direction. Map the host gateway in and use that:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
    # then point the app at http://host.docker.internal:32400
```

Everything else in the stack is a normal bridge service, so `http://<service
name>:<container port>` works as expected between them.

### One that is a scheduled job

`kometa` is the odd one: no port, no hostname, no healthcheck, nothing to browse.
Two things follow that are easy to get wrong.

- **Do not add a healthcheck.** A scheduler sitting idle between runs answers
  nothing, so any check marks it unhealthy forever.
- **Do not make it run once.** A container that runs and exits, paired with
  `restart: unless-stopped`, is a restart loop that hammers the Plex API. Let the
  image's own scheduler own the timing and leave the container up.

Its output is the Plex library and `docker compose logs kometa`, so that is where
you look to know whether it worked.

### "Plex addons" are not a thing any more

Plex removed its plugin framework in 2018, so nothing gets added *inside* the
Plex container. Everything in that ecosystem now — Jellyseerr, Overseerr,
Tautulli, Kometa — is a separate app talking to Plex over its HTTP API, which is
why they all arrive as containers and go through this section.

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

# infrastructure change (edited compose.yaml or .env)
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
| transcodes stay software-only despite `/dev/dri` | Plex Pass missing, or the Transcoder setting not ticked | §8 |
| `/dev/dri` present on the host, absent in the container | `docker compose` was given `-f`, or `COMPOSE_FILE` omits `hw/x86.yaml` | run bare `docker compose` from `~/stack` (§3) |
| `/` is ~100 GB on a 500 GB disk | Ubuntu's guided LVM left the VG unallocated | `lvextend -l +100%FREE` + `resize2fs` (§1) |
| a published port is reachable despite a ufw rule | Docker's rules sit ahead of ufw's chains | delete the `ports:` entry; ufw cannot close it (§2) |
| every container restarted overnight | `unattended-upgrades` updated docker-ce | `apt-mark hold` the docker packages (§2) |
| `conflicting options: port publishing and the container type network mode` | a `ports:` block on a service using `network_mode: service:gluetun` | move the port to gluetun's `ports:` |
| `port is already allocated` | host port collision | pick a free host port, update §4 |
| Radarr/Sonarr cannot reach qBittorrent or the indexers | namespace tenants have no DNS name | `http://gluetun:8080` and `http://gluetun:9696`, never their own names (§5) |
| every torrent errors the moment it is added | the default save path is still the image's `/downloads`, which this stack does not mount — the only media mount is `/data` | *Options → Downloads → Default Save Path* → `/data/incomplete` (§8) |
| torrents error on a path that does exist | `/mnt/active` is not writable by the container user | `sudo chown -R 1000:1000 /mnt/active`; `docker compose exec qbittorrent touch /data/probe` proves it either way |
| one service is unreachable while its namespace-mate answers | that container was never created — an earlier `up` aborted partway on a name conflict | `docker compose ps -a`; §12 clears the conflict, then `up -d` again |
| every app looks factory-fresh after a `git pull` | the appdata regrouping moved the mount paths, so the containers are pointed at new empty directories | **nothing is lost** — the data is still at `/srv/appdata/<service>`; do the regrouping in §15b |
| Prowlarr's qBittorrent client fails against host `gluetun` | Prowlarr is *inside* that namespace, and gluetun's DNS proxy replaced Docker's resolver | host `localhost`, port 8080 (§5) |
| HA discovers nothing it used to, no error anywhere | Plex's DLNA holds UDP 1900, or `avahi-daemon` holds 5353 | §8 — disable Plex's DLNA server, and avahi if unused |
| Zigbee2MQTT cannot open the adapter | the ConBee is on a different port, or deCONZ is running and holds it | `ls -l /dev/serial/by-id/`; the two are mutually exclusive (§8) |
| MQTT clients connect and are refused | mosquitto 2.x defaults to localhost-only, anonymous denied | its `config/mosquitto.conf` did not come across (§15d) |
| Prowlarr worked for weeks, then finds nothing | the TorrentDay cookie expired and the indexer was auto-disabled | redo §8 step 2; check *Indexers* for a disabled one |
| Prowlarr's indexer test returns a Cloudflare challenge | the cookie is bound to your browser's IP, not the VPN exit | FlareSolverr as an indexer proxy (§8) |
| `up -d` creates a network and reports no containers | the checkout directory name differs from the running containers' compose project | `COMPOSE_PROJECT_NAME=stack` in `.env` (§12) |
| `Conflict. The container name "/gluetun" is already in use` | same cause, seen from the other side: a second project trying to claim fixed `container_name`s | as above; `docker compose down` under the old project first |
| Prowlarr cannot reach `radarr:7878` when adding the App | gluetun's killswitch is dropping outbound traffic to the docker network | add `FIREWALL_OUTBOUND_SUBNETS=<the compose network subnet>` to gluetun; `docker network inspect stack_default` names it |
| `qBittorrent login failed` | temporary password rotated on restart | permanent password + subnet whitelist (§8) |
| upload is always 0 and the status bar shows *No direct connections* | NordVPN forwards no port, so nobody can connect in | structural — §8, "Why nothing uploads" |
| torrents stop by themselves shortly after finishing | a qBittorrent seeding limit, or Radarr removing completed downloads | §8; on a tracker paying for uptime, both are worth turning off |
| a download completes and Radarr never imports it | the download client and Radarr disagree about the path | both must mount `/mnt/active` as `/data` (§8); *Activity → Queue* names the path it tried |
| imports copy instead of hardlinking, and the disk fills twice | `downloads/` and `library/` are on different filesystems, or *Use Hardlinks* is off | both live under `/mnt/active` (§1); confirm with `ls -l` showing link count 2 on an imported file |
| Bazarr's library is empty | Radarr/Sonarr not connected, or connected with the wrong port | `radarr:7878`, `sonarr:8989` — bridge names, not `gluetun` (§8) |
| Jellyseerr requests approve but nothing downloads | *Settings → Services* has no Radarr/Sonarr, or the wrong root folder | §8 |
| `502`/`530` from a public hostname | cloudflared cannot resolve or reach the service | service name and **container** port; is it in the same project? |
| Cloudflare hostname 404s all assets | a Path prefix was set | leave Path empty, use a subdomain |
| `docker compose exec qbittorrent curl ifconfig.me` shows the ISP IP | namespace sharing not in effect | check `network_mode`, recreate |
| Ubuntu stuck at boot with no network | a USB SSD did not mount | `nofail` in `/etc/fstab` (§1) |
| that same command hangs | gluetun killswitch, usually DNS | `docker compose logs gluetun` |
| Plex shows half-finished files | downloading straight into a library | set the incomplete path (§8) |
| finished torrents take minutes to "move" | a category path escaped the `/data` mount | every category lives under `/data`, one disk (§1, §8) |
| Plex cannot delete a file | `/data` and `/archive` are mounted read-only | intended; delete through Radarr or Sonarr (§8) |
| moved a folder to archive and seeding stopped | qBittorrent only mounts `/mnt/active` | expected — the trade described in §1 |
| the active disk filled up with no warning | the finder's free-space guard is gone | *Minimum Free Space* in Media Management, plus `df` in your routine (§8) |
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

## 15. Migrating an existing setup

Two separate jobs, and they can happen months apart: restructuring the disks into
the layout in §1, and moving the box. Do the restructure first, on the Pi, so the
new layout is proven before new hardware is in the way.

| Subsection | What it covers |
| --- | --- |
| §15a | which branch you are on and how to keep it cheap to rebase |
| **§15b** | **the disks: swap the mountpoints, then rename directories, then lift container state off them** |
| §15c | telling the apps where it went: qBittorrent's paths, then Library Import |
| §15d | folding in the existing Home Assistant setup |
| §15e | carrying it all to the mini PC |

§15b and §15c are one sitting, in that order — 15b moves bytes, 15c makes the
applications agree with the result — and the stack stays down in between.

Within §15b the mountpoint swap comes first and is not optional: `/mnt/active`
and `/mnt/archive` do not exist yet, so every path in the steps after it is
wrong until fstab has been changed and the drives remounted.

### 15a. Run this on the Pi now — the `pi` branch

```bash
cd ~/stack && git fetch && git checkout pi
```

The `pi` branch differs from `main` in this file only — the hardware, dependency,
setup and troubleshooting sections — plus the default `COMPOSE_FILE` in
`.env.example`. Every compose file is byte-identical, because the real hardware
difference is the absent Quick Sync overlay, which is one line in `.env` (§3):

```dotenv
COMPOSE_FILE=compose.yaml
```

`RENDER_GID` stays blank there; nothing on the Pi reads it.

### 15b. Restructuring the disks

Three steps, in this order. The first one is the one that is easy to skip, and
nothing below works without it — every path in step 2 is a *new* mountpoint.

**1. Swap the mountpoints.** Which drive becomes which is not arbitrary: `active`
must be the one that already holds the torrent folders — the old `/mnt/ssd2` —
because then the bulk of the data never moves at all.

```bash
cd ~/stack && docker compose down     # containers hold the mounts busy
lsblk -f                              # note both UUIDs; fstab needs them
```

Now edit `/etc/fstab`. Replace the two `/mnt/ssd` and `/mnt/ssd2` lines with
these, using the UUIDs from `lsblk -f`:

```fstab
UUID=xxxx-xxxx  /mnt/active   ext4  defaults,noatime,nofail,x-systemd.device-timeout=10  0  2
UUID=yyyy-yyyy  /mnt/archive  ext4  defaults,noatime,nofail,x-systemd.device-timeout=10  0  2
```

#### On the Pi only: the `/srv/appdata` bind mount

**Skip this on the mini PC.** There, `/srv/appdata` is an ordinary directory on
the internal SSD and gets no fstab entry at all.

The Pi has no internal disk, and every compose file names `/srv/appdata`. Left as
an ordinary directory, that path is on the SD card — so the Plex library database
and every app's SQLite file would sit on the one device §1 says must hold nothing
but the OS. A **bind mount** fixes it by making an existing directory appear at a
second path: `/srv/appdata` becomes another name for a folder on the archive
drive, and the compose files work unchanged on both machines.

Both directories must exist before the mount can work:

```bash
sudo mkdir -p /mnt/archive/appdata /srv/appdata
```

Then add this to `/etc/fstab`, **below** the `/mnt/archive` line:

```fstab
/mnt/archive/appdata  /srv/appdata  none  bind,nofail,x-systemd.requires-mounts-for=/mnt/archive  0  0
```

A bind line looks nothing like a normal one, so field by field:

| Field | Value | Why |
| --- | --- | --- |
| source | `/mnt/archive/appdata` | the real directory, on the USB drive |
| target | `/srv/appdata` | the second name it appears under |
| type | `none` | a bind mount has no filesystem of its own |
| options | `bind` | says this is a bind rather than a device |
| | `nofail` | do not wedge boot if the drive is absent |
| | `x-systemd.requires-mounts-for=/mnt/archive` | do not attempt it until the archive drive is actually mounted |
| dump / pass | `0  0` | nothing to dump, nothing to fsck |

**Order matters, twice.** `mount -a` walks the file top to bottom, and at boot
systemd uses `requires-mounts-for` to know the dependency. Get either wrong and
the bind is attempted while `/mnt/archive` is still empty — which succeeds, on the
SD card, silently. Hence:

```bash
findmnt /srv/appdata     # must name the USB device, not the card
```

One trap: if `/srv/appdata` already contains files — from an earlier run of §7,
say — mounting over it *hides* them rather than merging. Move them aside first, or
they stay invisible until you unmount again.

```bash
sudo umount /mnt/ssd /mnt/ssd2
sudo mkdir -p /mnt/active /mnt/archive
sudo mount -a
findmnt /mnt/active /mnt/archive      # confirm both, before touching any data
ls /mnt/active                        # expect share/  — the torrent drive
ls /mnt/archive                       # expect media/  — the hand-managed one
```

If those two `ls` outputs are the other way round, the UUIDs are swapped in
fstab. Fix that now; every command in step 2 assumes this mapping.

`umount: target is busy` means something outside Docker holds the drive. `sudo
fuser -vm /mnt/ssd2` names it — and given the `share/` directory, a Samba export
is the likely answer, in which case its path in `/etc/samba/smb.conf` needs
updating to match the new mountpoint too.

```bash
sudo rmdir /mnt/ssd /mnt/ssd2         # do not leave these behind
```

That last line matters more than it looks. An empty `/mnt/ssd` left in place is a
directory on the root filesystem, so if a mount ever fails, everything that
thought it was writing to the USB drive quietly fills the boot disk instead.

**2. Move the data.** All renames within one filesystem, so all instant
regardless of how much is in them.

```bash
# active = the old /mnt/ssd2. Its finished torrent folders become downloads/,
# because that is where torrents seed from; Radarr and Sonarr will hardlink
# them into library/ in 15c, which costs no space.
mkdir -p /mnt/active/downloads /mnt/active/library
mv /mnt/active/share/torrents/movies      /mnt/active/downloads/movies
mv /mnt/active/share/torrents/tv          /mnt/active/downloads/tv
mv /mnt/active/share/torrents/incomplete  /mnt/active/incomplete
rmdir /mnt/active/share/torrents /mnt/active/share
mkdir -p /mnt/active/library/{movies,tv} \
         /mnt/active/books/{audiobooks,ebooks,magazines} /mnt/active/other

# archive = the old /mnt/ssd: the hand-managed library, which no *arr app
# manages. Plex reads it directly and it stays as it is.
mv /mnt/archive/media/movies  /mnt/archive/movies
mv /mnt/archive/media/tv      /mnt/archive/tv
```

**3. Lift the container state off the media drives.**

```bash
mkdir -p /srv/appdata/{media,home,apps}
mv /mnt/archive/qbittorrent-config  /srv/appdata/media/qbittorrent
mv /mnt/archive/prowlarr-config     /srv/appdata/media/prowlarr
mv /mnt/archive/gluetun             /srv/appdata/media/gluetun
mv /mnt/active/plex/config          /srv/appdata/media/plex/config
cp -r ~/git/matte-vm/data/.         /srv/appdata/apps/matte-vm/

# the one real copy: `other` was on the wrong disk (see below)
mv /mnt/archive/media/other/* /mnt/active/other/ && rmdir /mnt/archive/media/other
sudo chown -R 1000:1000 /srv/appdata /mnt/active /mnt/archive
```

On the Pi, `/srv/appdata` is the bind mount from step 1 — so `findmnt
/srv/appdata` should show the USB device before you run any of these, or they
land on the SD card and the whole point is lost.

**Already running with a flat `/srv/appdata`?** The grouping in §1 came later, so
an earlier setup has thirteen directories side by side rather than three. Moving
them is a rename within one filesystem, so it is instant and safe — but the stack
must be down, because a running container holds its bind mount:

```bash
cd ~/stack && docker compose down
cd /srv/appdata
sudo mkdir -p media home apps
sudo mv gluetun qbittorrent prowlarr radarr sonarr bazarr plex media/ 2>/dev/null
sudo mv jellyseerr kometa matte-vm apps/                              2>/dev/null
sudo mv homeassistant mosquitto zigbee2mqtt home/                     2>/dev/null
cd ~/stack && docker compose config -q && docker compose up -d
```

`2>/dev/null` because not every directory exists on every machine — an app you
have not set up yet is simply absent, and `mv` complaining about it is noise
rather than a problem. `config -q` before `up` catches a path you missed.

That `mv` of `other` is a genuine cross-disk copy, and it is fixing a bug rather
than creating one: `other` used to save to `/downloads/other` on the *library*
disk while qBittorrent's incomplete directory sat on the *torrent* disk, so every
completed `other` download was already being copied between disks — the exact
thing §1 forbids, on the one category nobody watches. Under `/data` it cannot
happen again.

Whatever is already in `other/` predates the split, so sort anything you want
into `books/` by hand once; the rest can stay where it is.

Do not start the stack yet — §15c is the other half of the same sitting.

### 15c. Handing the existing library to Radarr and Sonarr

Radarr and Sonarr have never seen any of this content, and qBittorrent's
container paths have changed under it. How much work that is depends entirely on
one question: **do you care whether the already-downloaded films keep seeding?**

#### If you do not — the simple path

Put the existing content straight into `library/`, adopt it there, and let the old
torrents go. This is a directory rename, so it is instant regardless of size.
Amend step 2 of §15b to move the folders one level differently:

```bash
mv /mnt/active/share/torrents/movies  /mnt/active/library/movies
mv /mnt/active/share/torrents/tv      /mnt/active/library/tv
mkdir -p /mnt/active/downloads/{movies,tv}      # start empty; only new grabs land here
```

Then, with Radarr and Sonarr configured (§8):

1. In qBittorrent, select the old torrents → **Remove** → *without* deleting
   files. They would otherwise sit there reporting missing data forever.
2. *Movies → Library Import* in Radarr, pointed at `/data/library/movies`, and
   *Series → Library Import* in Sonarr at `/data/library/tv`. Importing from
   inside its own root folder means Radarr adopts and renames the files **in
   place** — no copy, no hardlink, no second set of bytes.
3. Correct whatever it mismatched. That is the entire cost.

`downloads/` then only ever contains things downloaded from here on, which are
the ones that hardlink into `library/` and do keep seeding.

#### If you do — the seeding-preserving path

Keep §15b as written, so the finished folders become `downloads/`, and repoint
qBittorrent at them before importing.

Running torrents report missing files until they are told where the data went
(`/auto_movies` → `/data/downloads/movies`). There is no bulk rewrite, but
per-category is close enough:

1. Start the stack, open the WebUI, click a category in the sidebar.
2. Select all (Ctrl-A) → right-click → **Set location** → the new path.
3. It rechecks — fast, since the files are present and unmoved.

Or add compatibility mounts to `media/compose.yaml` so both spellings resolve,
migrate at leisure, then delete them:

```yaml
      - /mnt/active/downloads/movies:/auto_movies
      - /mnt/active/downloads/tv:/auto_tv
      - /mnt/active/incomplete:/auto_incomplete
```

Then *Library Import* pointed at `/data/downloads/movies` and
`/data/downloads/tv`. Importing from outside the root folder makes Radarr
**hardlink** into `library/`, so nothing is copied, the disk does not grow, and
the torrents carry on seeding from `downloads/` untouched.

#### Either way

Point Plex's libraries at `library/` and `/archive/...`, never at `downloads/`,
and remove the old folder paths from the library so nothing appears twice.

### 15d. Bringing Home Assistant in

It was running from its own compose project under `~/docker/homeassistant`, with
relative volumes (`./config`, `./mosquitto/...`). Those paths resolve against
whichever directory the compose file sits in, so simply moving the file would
silently point them at `~/stack/home/` — hence absolute `/srv/appdata` paths in
`home/compose.yaml`, and hence a data move.

**This is reversible, with one exception.** Everything is a bind mount, `rsync`
copies rather than moves, and the old compose file stays where it is — so the old
stack can be started again at any point. The exception is the Home Assistant
*version*: HA migrates `.storage` and its database forward on first start with a
newer image, and that migration does not roll back. So:

> **Do not `docker compose pull homeassistant` before the first run.** The image
> `ghcr.io/home-assistant/home-assistant:stable` is already on this machine, and
> Docker reuses a local image rather than fetching a newer one unless told to.
> Same image, same version, no migration, fully reversible. Pull it later, once
> the new arrangement has proved itself and you have a backup.

Take one anyway, from HA's own UI — *Settings → System → Backups → Create
backup* — and a tarball of the directory, before touching anything:

```bash
tar czf ~/ha-backup-$(date +%F).tgz -C ~/docker homeassistant
```

Then:

```bash
cd ~/docker/homeassistant && docker compose down     # release the ConBee and the ports
cd ~/stack

sudo mkdir -p /srv/appdata/home/{homeassistant,mosquitto,zigbee2mqtt}
sudo rsync -a ~/docker/homeassistant/config/           /srv/appdata/home/homeassistant/
sudo rsync -a ~/docker/homeassistant/mosquitto/        /srv/appdata/home/mosquitto/
sudo rsync -a ~/docker/homeassistant/zigbee2mqtt/data/ /srv/appdata/home/zigbee2mqtt/

docker compose up -d homeassistant mosquitto zigbee2mqtt
docker compose logs -f homeassistant
```

Three details in those commands that are not decoration:

- **The trailing slashes matter.** `rsync -a src/ dst/` copies the *contents*
  including dotfiles. Everything you care about — the entity registry, the device
  registry, your dashboards, your auth tokens — lives in `config/.storage`, and a
  `cp -r config/* …` would silently leave all of it behind. That is the one
  mistake that loses the lights.
- **`-a` preserves ownership, and you should let it.** Do **not** `chown` these
  afterwards: the eclipse-mosquitto image runs as uid 1883, not 1000, so
  flattening the directories to `1000:1000` breaks its ability to write its data
  and log. If something does report a permission error, match what the old
  directory had (`ls -ln ~/docker/homeassistant/mosquitto/data`) rather than
  guessing.
- **`down` first, always.** The ConBee can only be claimed by one container, and
  1883, 8082 and 8123 can only be bound once.

Four things to check, in this order, because each one masks the next:

1. **Mosquitto accepted its config** — `docker compose logs mosquitto` should show
   it opening the listener, not falling back to localhost-only.
2. **Zigbee2MQTT found the stick and the broker** — its log names the adapter and
   then `Connected to MQTT server`.
3. **Home Assistant came up on 8123** with your dashboards intact.
4. **Devices are still there.** Zigbee pairings live in
   `/srv/appdata/home/zigbee2mqtt/configuration.yaml` — the network key and PAN id in
   particular — not in the stick, so if that file came across nothing needs
   re-pairing.

**If any of that goes wrong**, the old setup is untouched:

```bash
cd ~/stack && docker compose stop homeassistant mosquitto zigbee2mqtt
cd ~/docker/homeassistant && docker compose up -d
```

Delete `~/docker/homeassistant` only after a few days of the new arrangement
behaving. Then do the two conflict fixes in §8 — Plex's DLNA and `avahi-daemon`
— or HA's discovery will be quietly broken in a way that looks like nothing at
all.

### 15e. Moving to the mini PC

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
  and the library *paths* that matter are container-side (`/data/library/movies`,
  `/archive/movies`), so they do not change with the hardware. No full rescan.
  They *did* change in 15b, though, so do that restructure once and let the mini
  PC inherit the finished shape.
- **Keep a `plex.tv/claim` token ready** in case the server shows as unclaimed;
  usually the machine identity in the config carries over and it does not.
- **Copy qBittorrent's config only while the container is stopped**, or it will
  be overwritten on exit. Its session survives, because by now the container
  paths are already the new ones. The same applies to Radarr's, Sonarr's and
  Bazarr's SQLite databases — all under `/srv/appdata`, all in the tarball.
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
