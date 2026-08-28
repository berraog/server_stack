# homelab runbook

Everything needed to rebuild this server from a blank disk, plus how to change
or extend it afterwards. The hardware is a BOSGAME mini PC (Intel N95) running
Ubuntu Server 24.04. The Raspberry Pi it replaces is still a supported target
on the `pi` branch; §15 covers restructuring the disks and moving between them. This file is the `stack` repo: the repo that
describes the machine and, through `compose.yaml`, assembles it.

**What runs here**

| Stack | Services | Reachable at |
| --- | --- | --- |
| media | gluetun (VPN), qBittorrent, Prowlarr, Plex | LAN ports |
| automation | Radarr, Sonarr, Bazarr | LAN ports only — admin tools |
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

```
/srv/appdata/          internal 500 GB SSD — all container state
  gluetun/  qbittorrent/  prowlarr/  plex/{config,transcode}/  matte-vm/

/mnt/active/           USB SSD — everything torrents touch
  incomplete/          qBittorrent's default save path AND its incomplete path
  downloads/movies/    qBittorrent's targets; torrents keep seeding from here
  downloads/tv/
  library/movies/      Radarr's and Sonarr's output; what Plex reads
  library/tv/
  books/               manual grabs — ebooks, audiobooks, magazines
  other/               manual grabs — anything else

/mnt/archive/          USB SSD — content moved off active once it fills
  movies/  tv/  ...    read-only; mirrors whichever active folders you move
```

**Why downloads and library are separate directories.** Radarr and Sonarr do not
leave a finished file where it landed; they *import* it into a library folder
they name and organise. To avoid storing every file twice that import is a
hardlink, which works only inside one filesystem — hence both trees under
`/mnt/active` rather than one of them on the internal SSD. The torrent keeps
seeding from `downloads/`, Plex reads the tidy names in `library/`, and the bytes
exist once.

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
  media/compose.yaml              gluetun, qbittorrent, prowlarr, plex
  media/arr.yaml                  radarr, sonarr, bazarr
  apps/matte-vm.yaml              one fragment per standalone app
  apps/jellyseerr.yaml            third-party image, same shape
  apps/kometa.yaml                scheduled job: no port, no hostname
  hw/x86.yaml, hw/pi.yaml         per-hardware overlays, picked in .env
  bin/autodeploy                  poll git, redeploy what changed
  autodeploy.conf                 repo -> service map
  systemd/autodeploy.{service,timer}   installed into /etc (§11)
  .env                            ALL secrets, gitignored
  .env.example                    same keys, no values, committed
  .gitignore
~/git/matte-vm/                   app repo: Dockerfile + its own compose file
~/git/torrent_finder/             retired from the stack (§8), repo kept
~/git/<next-app>/                 same shape
/srv/appdata/<service>/           container state, internal SSD (§1)
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
| 8090 | frontend (nginx) | container side is 80 |
| 7878 | Radarr | LAN only, never tunnelled |
| 8989 | Sonarr | LAN only, never tunnelled |
| 6767 | Bazarr | LAN only, never tunnelled |
| 5055 | Jellyseerr | its own default |
| 32400 | Plex | `network_mode: host` |

Container ports never collide, only host ports. An app reachable solely through
the tunnel needs no `ports:` entry at all, and a batch job like `kometa` needs
none either — it never listens.

## 5. The rule that breaks things first

`qbittorrent` and `prowlarr` use `network_mode: "service:gluetun"`, so they have
**no network identity of their own**:

- `http://qbittorrent:8080` never resolves. Use **`http://gluetun:8080`**.
- `http://prowlarr:9696` never resolves. Use **`http://gluetun:9696`**.
- That applies to every app configured to reach them, which is now most of the
  box: Radarr and Sonarr point their download client and indexer proxy at
  `gluetun`, and so does Prowlarr's own download client. Radarr, Sonarr and
  Bazarr are ordinary bridge services themselves, so *their* names do resolve —
  `http://radarr:7878` from Bazarr or Jellyseerr is correct.
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
| `NORDVPN_SERVICE_*` | NordVPN | gluetun |
| `CLOUDFLARE_TUNNEL_TOKEN` | Cloudflare | cloudflared |
| qBittorrent's WebUI login | qBittorrent's API | Radarr, Sonarr and Prowlarr, each in its own UI |
| Prowlarr's / Radarr's / Sonarr's API keys | each other | entered in each app's UI, never in `.env` |

## 7. First-time setup, in order

```bash
# 1. clone
mkdir -p ~/git && cd ~/git
git clone git@github.com:<you>/matte-vm.git
cd ~ && git clone git@github.com:<you>/stack.git       # this repo

# 2. secrets
cp ~/stack/.env.example ~/stack/.env && chmod 600 ~/stack/.env && nano ~/stack/.env

# 3. data directories, owned by the container user
#    internal SSD: container state.  USB drives: bulk media only.
sudo mkdir -p /srv/appdata/{gluetun,qbittorrent,prowlarr,plex/config,plex/transcode}
sudo mkdir -p /srv/appdata/{radarr,sonarr,bazarr,jellyseerr,kometa,matte-vm}
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
3. *Options → Downloads*:
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
cp /srv/appdata/qbittorrent/qBittorrent/qBittorrent.conf{,.bak}
sed -i '/^WebUI\\Password_PBKDF2=/d' /srv/appdata/qbittorrent/qBittorrent/qBittorrent.conf
docker compose up -d qbittorrent
docker compose logs qbittorrent | grep -i "temporary password"
# or: python3 ~/git/torrent_finder/deploy/raspberry-pi/qbt_password.py
#     and paste the line under [Preferences]
```

### Prowlarr (`http://<host>:9696`)

Prowlarr is the indexer proxy: it holds the TorrentDay credentials once, and
pushes the indexer definition out to Radarr and Sonarr so they never hold it.

1. *Settings → General → Security*: note the API key. Radarr, Sonarr and
   Jellyseerr each need it, entered in their own UIs.
2. *Indexers → Add Indexer → TorrentDay*, authenticate with your account cookie.
3. Run one search from Prowlarr's own *Search* tab before trusting it from code.
4. *Settings → Apps → Add → Radarr*, then again for *Sonarr*. Server addresses
   are `http://radarr:7878` and `http://sonarr:8989`; the Prowlarr server address
   it asks for in return is `http://gluetun:9696`, **not** `http://prowlarr:9696`
   (§5). Indexers then sync outward automatically.
5. *Settings → Download Clients → Add → qBittorrent*, host `gluetun`, port
   `8080`. This is what makes step 3's *Search* tab able to grab, which is how
   the occasional ebook or audiobook gets downloaded now — set the category to
   `books` or `other` on the grab.

That last step is the whole manual workflow. Prowlarr's search is not as pleasant
as a purpose-built UI, but it is already here, already authenticated, and already
inside the VPN namespace, and the alternative was maintaining an app for
something that happens a few times a year.

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
where, which is why it could not run here until now.

1. *Settings → Radarr*: address `radarr`, port `7878`, Radarr's API key. Same
   for *Settings → Sonarr* with `sonarr` and `8989`. These are bridge services,
   so their own names resolve — no `gluetun` here (§5).
2. *Settings → Languages*: define a profile (say Swedish, then English) and set
   it as default for both series and movies.
3. *Settings → Providers*: add OpenSubtitles or similar. An account is free and
   the anonymous limits are low enough to look broken.
4. *Settings → Subtitles*: leave **Use embedded subtitles** on so it does not
   fetch what a file already carries as text.

Once this works, turn Plex's own subtitle agent back off — two things fetching
subtitles for the same file produce duplicates that are tedious to unpick. Bazarr
writing `.srt` files next to the video is also strictly better for the burn-in
problem than an embedded PGS: a sidecar SRT is text, so clients render it and
nothing transcodes (§8).

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
sudo mkdir -p /srv/appdata/jellyseerr && sudo chown -R 1000:1000 /srv/appdata/jellyseerr
cd ~/stack && docker compose up -d jellyseerr
docker compose logs -f jellyseerr        # a permissions complaint here means the chown
```

Then in the setup wizard: choose **Plex**, and for the server address use
`host.docker.internal` port `32400` — not `plex`, which does not resolve (§10).
Sign in with your Plex account, import the Movies and TV libraries, and set
*Settings → General → Application URL* to `https://requests.bjorngreen.se`.

Then wire the fulfilment, which is what makes it more than an inbox —
*Settings → Services → Add Radarr Server*, and again for Sonarr:

| Field | Value |
| --- | --- |
| Hostname | `radarr` / `sonarr` — bridge services, so the names resolve (§5) |
| Port | `7878` / `8989` |
| API key | from that app's *Settings → General* |
| Quality profile | the 1080p profile from §8, so requests inherit the format policy |
| Root folder | `/data/library/movies` / `/data/library/tv` |

A request now goes: someone asks → you approve (or auto-approve trusted users) →
Radarr searches Prowlarr → qBittorrent downloads through the VPN → Radarr imports
and renames into `library/` → Plex scans → Jellyseerr marks it *Available*. No
step in that chain is yours once the profile is right.

**The intended flow: the Plex watchlist.** Plex has no extension mechanism, so
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
sudo mkdir -p /srv/appdata/kometa && sudo chown -R 1000:1000 /srv/appdata/kometa
# the image ships a fully commented template; take it and edit
docker run --rm kometateam/kometa:latest cat /config/config.yml.template \
  | sudo tee /srv/appdata/kometa/config.yml >/dev/null
sudo nano /srv/appdata/kometa/config.yml
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
      - /srv/appdata/<new-app>:/config
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
docker image prune -f

# gluetun / VPN change: recreates its tenants, interrupts torrents
docker compose pull gluetun && docker compose up -d gluetun

# state
docker compose ps
docker compose logs -f --tail=100 radarr
df -h / /mnt/active /mnt/archive
docker system df
```

Restart blast radius, worth internalising:

| Restarting | Also takes down |
| --- | --- |
| `gluetun` | qBittorrent, Prowlarr (namespace tenants) |
| `cloudflared` | every public hostname, ~10s |
| `qbittorrent`, `prowlarr` | only itself |
| `radarr`, `sonarr`, `bazarr`, an app | only itself |

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
| `qBittorrent login failed` | temporary password rotated on restart | permanent password + subnet whitelist (§8) |
| a download completes and Radarr never imports it | the download client and Radarr disagree about the path | both must mount `/mnt/active` as `/data` (§8); *Activity → Queue* names the path it tried |
| imports copy instead of hardlinking, and the disk fills twice | `downloads/` and `library/` are on different filesystems | both live under `/mnt/active` (§1) |
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
| **§15b** | **moving the data — mountpoints, directory renames, container state** |
| §15c | telling the apps where it went: qBittorrent's paths, then Library Import |
| §15d | carrying it all to the mini PC |

§15b and §15c are one sitting, in that order: 15b moves bytes, 15c makes the
applications agree with the result.

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

That `mv` is a genuine cross-disk copy, and it is fixing a bug rather than
creating one: `other` used to save to `/downloads/other` on the *library* disk
while qBittorrent's incomplete directory sat on the *torrent* disk, so every
completed `other` download was already being copied between disks — the exact
thing §1 forbids, on the one category nobody watches. Under `/data` it cannot
happen again.

Whatever is already in `other/` predates the split, so sort anything you want
into `books/` by hand once; the rest can stay where it is.

### 15c. Handing the existing library to Radarr and Sonarr

Two things need saying: qBittorrent's container paths changed, and Radarr and
Sonarr have never seen any of this content.

**qBittorrent first.** Running torrents will report missing files until they are
told where the data went (`/auto_movies` → `/data/downloads/movies`). There is no
bulk rewrite, but per-category is close enough:

1. Start the stack, open the WebUI, click a category in the sidebar.
2. Select all (Ctrl-A) → right-click → **Set location** → the new path.
3. It rechecks — fast, since the files are present and unmoved.

If you would rather not touch the session, add compatibility mounts to
`media/compose.yaml` so both spellings resolve, migrate at leisure, then delete
them:

```yaml
      - /mnt/active/downloads/movies:/auto_movies
      - /mnt/active/downloads/tv:/auto_tv
      - /mnt/active/incomplete:/auto_incomplete
```

**Then adopt the library.** With Radarr and Sonarr configured (§8), use
*Movies → Library Import* (Radarr) and *Series → Library Import* (Sonarr) pointed
at `/data/downloads/movies` and `/data/downloads/tv`. They match each folder to a
title, then import into `/data/library/...` by **hardlink** — so nothing is
copied, the disk does not grow, and the torrents carry on seeding from
`downloads/` untouched. Expect to correct a few mismatched titles by hand; that
is the whole cost.

Point Plex's libraries at `library/` afterwards, not `downloads/`, and remove the
old folders from the library so the same film does not appear twice.

The alternative is legitimate: let the Pi seed the old layout until it drains and
start clean. Finished files are already in `/mnt/archive`, so only in-flight
torrents are lost.

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
