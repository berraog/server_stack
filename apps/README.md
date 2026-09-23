# apps — one fragment per standalone application

Each file here is a single application wired into the machine by the root
[`compose.yaml`](../compose.yaml). An app keeps its own repo under `~/git`; this
directory owns only *how it is wired in*. Read
[the root README](../README.md) first for the disks, `.env` and the first-run
order.

| Fragment | Service | Reachable at |
| --- | --- | --- |
| [`jellyseerr.yaml`](jellyseerr.yaml) | Jellyseerr | `http://<host>:5055`, `requests.bjorngreen.se` |
| [`kometa.yaml`](kometa.yaml) | Kometa | nothing — scheduled job, no UI |
| [`matte-vm.yaml`](matte-vm.yaml) | matte-vm | its own Cloudflare hostname |

---

## Jellyseerr (`http://<host>:5055`, `requests.bjorngreen.se`)

Create the config directory first, or the container starts and cannot write:

```bash
sudo mkdir -p /srv/appdata/apps/jellyseerr && sudo chown -R 1000:1000 /srv/appdata/apps/jellyseerr
cd ~/stack && docker compose up -d jellyseerr
docker compose logs -f jellyseerr        # a permissions complaint here means the chown
```

**1. The wizard.** Choose **Plex**, and for the server address use
`host.docker.internal` port `32400` — not `plex`, which does not resolve (this file).
Sign in with your Plex account; the account that completes the wizard becomes the
Jellyseerr owner.

**2. Enable the libraries and scan.** *Settings → Plex → Libraries*: tick Movies
and TV Shows, then **Start Scan**. Skipping this is why availability detection
appears not to work — Jellyseerr only knows a request is fulfilled if it is
watching the library the file landed in. Set *Settings → General → Application
URL* to `https://requests.bjorngreen.se` at the same time, or Plex's OAuth
redirect and every notification link points at the wrong place.

**3. Wire the fulfilment**, which is what makes it more than an inbox.
*Settings → Services → Add Radarr Server*, then again for Sonarr. Field names
shift a little between Jellyseerr releases, but these are the values:

| Field | Radarr | Sonarr |
| --- | --- | --- |
| **Default Server** | **on** | **on** |
| 4K Server | off | off |
| Server Name | `Radarr` | `Sonarr` |
| Hostname or IP | `radarr` | `sonarr` |
| Port | `7878` | `8989` |
| Use SSL | off | off |
| API Key | Radarr → *Settings → General* | Sonarr → *Settings → General* |
| URL Base | blank | blank |
| Quality Profile | your 1080p profile ([media/README.md](../media/README.md)) | your 1080p profile |
| Root Folder | `/data/library/movies` | `/data/library/tv` |
| Minimum Availability | `Released` | *(not shown)* |
| Season Folders | *(not shown)* | on |
| External URL | `http://<pi-ip>:7878` | `http://<pi-ip>:8989` |
| **Enable Scan** | **on** | **on** |
| **Enable Automatic Search** | **on** | **on** |

Six of those are worth understanding rather than copying:

- **Default Server** — nothing routes without it, and the failure is silent:
  requests approve and then sit there forever.
- **Hostname** is the service name, not `gluetun` and not an IP. Radarr and
  Sonarr are ordinary bridge containers, so Jellyseerr resolves them directly
  ([the gluetun namespace rule](../media/README.md#the-rule-that-breaks-things-first)).
- **Quality Profile and Root Folder are dropdowns that stay empty until the
  connection works.** That is your connection test: if they will not populate,
  the API key or the hostname is wrong, and nothing below matters yet.
- **Enable Automatic Search** — without it Radarr adds the film, monitors it, and
  never looks for it. The request sits in Jellyseerr as approved and in Radarr as
  wanted, with nothing in between, which is a confusing way to spend an evening.
- **Minimum Availability `Released`** stops Radarr hunting for films that are not
  out yet. Set it to `Announced` only if you want it grabbing the moment anything
  appears, which on a private tracker mostly means grabbing cam rips.
- **External URL** is what the *Open in Radarr* link points at. The internal name
  `radarr` means nothing to your browser, so use the box's LAN address — or leave
  it blank and accept a dead link.

Leave the 4K fields alone. They are for a second Radarr managing a separate 4K
library, which [media/README.md](../media/README.md)'s release policy deliberately does not have. Sonarr may also
offer separate anime profile and root folder settings; leave them matching the
defaults unless you want anime filed apart.

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
that torrent-finder used to enforce is gone ([media/README.md](../media/README.md)) and automation fills a disk
quickly.

**5. Get told when something is requested.** *Settings → Notifications* — Discord
or email is enough. Without it, a request that needs approval waits for you to
happen to open the site, which rather defeats the point of the inbox.

A request now goes: someone asks → you approve (or auto-approve trusted users) →
Radarr searches Prowlarr → qBittorrent downloads through the VPN → Radarr imports
and renames into `library/` → Plex scans → Jellyseerr marks it *Available*. No
step in that chain is yours once the profile is right.

**6. The intended flow: the Plex watchlist.** Plex has no extension mechanism, so
there is no request button in the Plex apps (this file). What there is, is Plex's own
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
you are the only requester — see [root §9](../README.md) for why it stops being right the moment
anyone else is.

## Kometa (no UI — `docker compose logs kometa`)

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
| `plex.url` | `http://host.docker.internal:32400` — **not** `http://plex:32400` (this file) |
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
real escape hatch if a run makes a mess is the Plex database from the [root §12](../README.md) backup,
which is heavy but genuine, so take one before the first overlay run.

Nothing here is on a hostname or a port. If Kometa appears to do nothing, it is
almost always the schedule: `KOMETA_TIME` and the container's timezone decide
when it wakes, and the logs say what it did.

---

## Adding a new app

#### One you build from a repo

The app repo needs nothing but a `Dockerfile`; the stack owns how it is wired in.

1. Clone it into `~/git/<new-app>`.
2. Write `~/stack/apps/<new-app>.yaml` from the skeleton below — one service,
   `container_name` set so the tunnel URL is predictable, state under
   `/srv/appdata/apps/<new-app>`, and a host port from [root §4](../README.md) only if you want LAN
   access.
3. Add it to `~/stack/compose.yaml` under `include:`.
4. Add a line to `~/stack/autodeploy.conf`.
5. Add any secrets to `~/stack/.env` **and** `.env.example`, referenced with
   `${VAR:?set in ~/stack/.env}` so a missing value fails loudly at start.
6. Add the public hostname in Cloudflare ([root §9](../README.md)) plus an Access policy.
7. Update the port registry in [root §4](../README.md) and the table at the top of this file.

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

#### One that ships as an image

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
`http://plex:32400` does not resolve — the same shape of trap as [the gluetun namespace rule](../media/README.md#the-rule-that-breaks-things-first), from the
other direction. Map the host gateway in and use that:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
    # then point the app at http://host.docker.internal:32400
```

Everything else in the stack is a normal bridge service, so `http://<service
name>:<container port>` works as expected between them.

#### One that is a scheduled job

`kometa` is the odd one: no port, no hostname, no healthcheck, nothing to browse.
Two things follow that are easy to get wrong.

- **Do not add a healthcheck.** A scheduler sitting idle between runs answers
  nothing, so any check marks it unhealthy forever.
- **Do not make it run once.** A container that runs and exits, paired with
  `restart: unless-stopped`, is a restart loop that hammers the Plex API. Let the
  image's own scheduler own the timing and leave the container up.

Its output is the Plex library and `docker compose logs kometa`, so that is where
you look to know whether it worked.

#### "Plex addons" are not a thing any more

Plex removed its plugin framework in 2018, so nothing gets added *inside* the
Plex container. Everything in that ecosystem now — Jellyseerr, Overseerr,
Tautulli, Kometa — is a separate app talking to Plex over its HTTP API, which is
why they all arrive as containers and go through this section.
