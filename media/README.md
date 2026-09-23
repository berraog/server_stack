# media — torrents, indexers, the \*arr apps and Plex

The download and library half of the box: [`compose.yaml`](compose.yaml) holds
gluetun, qBittorrent, Prowlarr and Plex; [`arr.yaml`](arr.yaml) holds Radarr,
Sonarr and Bazarr. Both are assembled into the machine by the root
[`compose.yaml`](../compose.yaml).

Read [the root README](../README.md) first for the disks, `.env` and the
first-run order. This file is what you configure *after* the containers are up.

| Service | LAN address | Published by |
| --- | --- | --- |
| qBittorrent | `http://<host>:8080` | gluetun |
| Prowlarr | `http://<host>:9696` | gluetun |
| Radarr | `http://<host>:7878` | itself |
| Sonarr | `http://<host>:8989` | itself |
| Bazarr | `http://<host>:6767` | itself |
| Plex | `http://<host>:32400/web` | host network |

---

## How the storage model works

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
(the per-service sections below) is what leaves the original name in place. If it ever *cannot* hardlink —
because the two trees ended up on different filesystems — it silently falls back
to copying: seeding still works, but every film now genuinely occupies twice its
size. A library growing about twice as fast as it should is the symptom, and [root §13](../README.md)
has it.

**One caveat on not minding about seeding.** TorrentDay is a private tracker, and
private trackers generally enforce a ratio and hit-and-run rules, so dropping
every torrent the moment it finishes is how accounts get disabled — check its own
rules. Never seeding at all is a different decision, and the arrangement above is
precisely what makes seeding cost nothing: the bytes Plex is serving are the same
bytes you are seeding.

**`books/` and `other/` are the manual half.** Radarr covers movies, Sonarr
covers TV, and nothing here covers an occasional ebook or audiobook — so those
are grabbed by hand from Prowlarr's own Search tab (the per-service sections below) into their own
qBittorrent categories. Inside `books/`, keep `audiobooks/`, `ebooks/` and
`magazines/` as subfolders if you want them apart. Nothing on this box reads
them; that would be Calibre-web or Audiobookshelf, added per [apps/README.md](../apps/README.md).

---

## The rule that breaks things first

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
  active torrent. Never let a routine app deploy touch it (see [apps/README.md](../apps/README.md)).

---

## Before you configure anything

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
by design ([root §9](../README.md)), so *Authentication Required → Disabled for Local Addresses* is a
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

**Adding something to watch.** Three ways in, and only the last one involves
Radarr or Sonarr directly:

| Route | Where | Good for |
| --- | --- | --- |
| **Plex watchlist** | inside the Plex app you already use | the default. Add to Watchlist and it arrives; no second site, nothing to learn (the per-service sections below) |
| **Jellyseerr** | `requests.bjorngreen.se`, from anywhere | one search box covering both films and series, and the only route other people get |
| Radarr / Sonarr | LAN only | when you want *that specific release* |

**You do not search Radarr and Sonarr separately in normal use.** Jellyseerr
searches one TMDB catalogue covering both, works out whether a result is a film
or a series, and hands it to the right one — that is what it is for, and it is
why it is the only one of the three on the tunnel.

Going to Radarr or Sonarr directly is for control rather than discovery. Their
**Interactive Search** — the magnifying glass on a film or season — lists the
actual releases with size, seeders and quality, and lets you take one. That is
the tool for overriding a bad automatic grab, or forcing a 1080p WEB-DL when the
profile picked something that burns in subtitles (the per-service sections below). Jellyseerr cannot do this:
it requests, and Radarr chooses per profile.

**Do not put Radarr or Sonarr on the tunnel to unify them.** They cannot share a
hostname anyway — [root §9](../README.md) covers why a path prefix breaks these apps — and combining
them is precisely the problem Jellyseerr already solves, without exposing two
applications that can delete your library ([root §14](../README.md)).

---

## qBittorrent (`http://<host>:8080`)

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
   app imports it — which is the whole point of the split ([root §1](../README.md)).

Every path above is under the single `/data` mount, which is the whole active
disk ([root §1](../README.md)). That is what makes completion a rename and an import a hardlink, and
it is why `incomplete/` is a sibling rather than living inside a library folder:
incomplete files inside a Plex library get scanned half-written and pollute it.

**`/data` must be spelled identically everywhere.** qBittorrent, Radarr, Sonarr
and Bazarr all mount `/mnt/active` at `/data` for one reason: qBittorrent reports
a finished torrent's location as a path, and Radarr has to open that exact path
to import it. Mount it as `/downloads` in one and `/data` in another and imports
simply never happen, with no error that names the cause ([root §13](../README.md)).

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
  [root §14](../README.md) keeps router ports closed on purpose.

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

That last row is worth dwelling on, because it interacts with the hardlinks ([root §1](../README.md)).
Keeping a torrent seeding keeps the `downloads/` name alive alongside the
`library/` one — which costs nothing, since they are the same inode. There is no
disk-space reason to remove torrents early, and on a tracker that pays you for
uptime there is a reason not to.

## Prowlarr (`http://<host>:9696`)

Prowlarr is the indexer proxy: it holds the TorrentDay credentials once, and
pushes the indexer definition out to Radarr and Sonarr so they never hold it.

Steps 1–3 stand alone and are worth doing first. Steps 4–5 talk to other
containers, so they need Radarr and Sonarr already running with their API keys,
and qBittorrent already carrying a permanent password (the per-service sections below) — come back for them.

**1. Note the API key.** *Settings → General → Security*, or straight out of
`config.xml` (the per-service sections below opening). Radarr, Sonarr and Jellyseerr each want it, entered in
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
| Prowlarr Server | `http://gluetun:9696` | Prowlarr has no network identity of its own — [the gluetun namespace rule](#the-rule-that-breaks-things-first) |
| API Key | from that app's *Settings → General* | |

Getting the second one wrong is the usual mistake: it is the address *Radarr*
will use to call back to Prowlarr, so it must be gluetun's. Indexers then sync
outward automatically, and Radarr's *Settings → Indexers* fills in by itself.

**5. Add qBittorrent as a download client.** *Settings → Download Clients → Add
→ qBittorrent*, host **`localhost`**, port `8080`, the WebUI login from the per-service sections below.

Not `gluetun` — Prowlarr and qBittorrent share a network namespace, so they are
already on the same loopback, and `gluetun` does not resolve from inside it
([the gluetun namespace rule](#the-rule-that-breaks-things-first)). Radarr and Sonarr *do* use `gluetun` for the same client, because they are
ordinary bridge containers reaching in from outside. Same download client, two
different addresses, depending on which side is asking.

One consequence of using loopback: the subnet whitelist from the per-service sections below covers
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
`/mnt/active/other`, the `/data` mount is not what you think it is (the per-service sections below). If step
2 returns the same address twice, stop and fix the VPN before downloading
anything else — that is the one failure worth catching immediately.

**Two things that will fail later, not now.** The cookie expires — silently, weeks
in, and Prowlarr disables an indexer after enough consecutive failures, so
"nothing finds anything any more" usually means repeating step 2. And if TorrentDay
puts Cloudflare in front of itself, a cookie minted in your browser is bound to
your browser's address and will not work from the VPN exit; the standard answer is
a FlareSolverr container, added per [apps/README.md](../apps/README.md) and pointed at from *Settings → Indexers →
Indexer Proxies*.

## Radarr (`http://<host>:7878`) and Sonarr (`http://<host>:8989`)

Identical setup; do both. Neither is on the tunnel — they can rewrite your
library, so they stay on the LAN ([root §9](../README.md)).

1. *Settings → Media Management → Root Folders → Add*:
   `/data/library/movies` for Radarr, `/data/library/tv` for Sonarr.
2. *Settings → Media Management*: turn on **Rename Movies** / **Rename
   Episodes**, and leave **Use Hardlinks instead of Copy** on. If it ever copies
   instead, `/data` is two filesystems and [root §1](../README.md) has been violated.
3. *Settings → Download Clients → Add → qBittorrent*: host `gluetun`, port
   `8080`, the WebUI login from the per-service sections below. Leave *Category* at `radarr` / `sonarr` — it
   creates them in qBittorrent for you.
4. *Settings → Indexers*: already populated by Prowlarr (the per-service sections below). If empty, step 4
   of Prowlarr did not take.
5. *Settings → Profiles → Quality Profiles*: this is where the buffering fix
   from the per-service sections below's release guidance stops being a habit. Build a profile that allows
   1080p WEB-DL and Bluray, rejects Remux and 2160p, and it will never again
   grab a file that forces a burn-in transcode.
6. *Settings → Profiles → Quality Definitions*: set the maximum sizes. This
   replaces the `MAX_TORRENT_SIZE_GB` guard the finder used to enforce.

#### When a download completes and nothing gets imported

**Prowlarr is not involved.** Its job ends the moment indexers are synced into
Radarr and Sonarr; after that Radarr talks to the indexer directly, using the
definition Prowlarr handed it, and to qBittorrent directly. Nothing about
importing, renaming or moving passes through Prowlarr, so an import that never
happens is a Radarr or Sonarr question.

**The exception, and it looks exactly like this failure:** a torrent grabbed from
**Prowlarr's own Search tab** is invisible to Radarr. Prowlarr sets its own
category — whatever you picked on the grab — and Radarr only watches the
`radarr` category for downloads *it* started. Nothing imports because as far as
Radarr is concerned nothing was ever requested. Prowlarr's search is for the
manual long tail (the per-service sections below); anything Radarr should own must be searched **from
Radarr**, under the film's own page or *Movies → Add New*.

**Where the reason is written:** *Activity → Queue* in Radarr. A stuck import
carries its explanation inline, and it is almost always one of these:

| Queue says | Cause | Fix |
| --- | --- | --- |
| the download is not listed at all | Radarr never grabbed it — see the Prowlarr trap above, or the category is wrong | *Settings → Download Clients → qBittorrent → Category* must be `radarr`, and that category must exist in qBittorrent |
| *No files found are eligible for import* | the release is a sample, an archive, or the wrong content | manual import, or blocklist and let Radarr try another release |
| a path Radarr cannot access | qBittorrent and Radarr disagree about `/data` | both must mount `/mnt/active` at `/data` ([root §1](../README.md)). Leave *Remote Path Mappings* **empty** — filling it in hides the real fault |
| nothing, but the file stays in `downloads/` | *Completed Download Handling* is off | *Settings → Download Clients → Completed Download Handling → Enable* |
| an import error mentioning permissions | `/data/library/...` is not writable by the container user | `sudo chown -R 1000:1000 /mnt/active` |

Prove the path agreement directly rather than reasoning about it — the same file
must be visible from both sides at the same path:

```bash
cd ~/stack
docker compose exec qbittorrent ls -la /data/downloads/movies
docker compose exec radarr      ls -la /data/downloads/movies    # must match
docker compose exec radarr      touch /data/library/movies/.probe && echo writable
```

If the first two disagree, that is the whole problem and no Radarr setting will
fix it. If they agree and the import still fails, the Queue's own message is the
next thing to read.

**One guard did not survive the move.** The finder refused a download when free
space fell below `MIN_FREE_SPACE_GB`, measured on the download disk. Radarr and
Sonarr have a *Minimum Free Space* under *Media Management* (advanced), but it
defaults to a value far below 20 GB and only blocks the import, not the download.
Set it, and keep `df -h /mnt/active` in your routine ([root §12](../README.md)) — automation fills a
disk considerably faster than choosing releases by hand did.

## Bazarr (`http://<host>:6767`)

Bazarr has no library of its own: it asks Radarr and Sonarr what exists and
where, which is why it could not run here until now. Set it up after both of
those have imported your library, or it will connect to two empty catalogues and
look broken.

**1. Connect Radarr and Sonarr.** *Settings → Radarr* — address `radarr`, port
`7878`, Radarr's API key; then *Settings → Sonarr* with `sonarr` and `8989`.
These are ordinary bridge services, so their own names resolve; no `gluetun`
here ([the gluetun namespace rule](#the-rule-that-breaks-things-first)).

**Leave the path mapping table empty.** It exists because Bazarr is usually told
a path by Radarr that means something different inside Bazarr's own container,
and filling it in wrongly is the classic way to break this. It is unnecessary
here for a specific reason: qBittorrent, Radarr, Sonarr and Bazarr all mount
`/mnt/active` at the same container path, `/data` ([root §1](../README.md)), so a path Radarr reports
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
nothing transcodes, where an embedded PGS forces burn-in (the per-service sections below).

Bazarr writes those `.srt` files into `/data/library/...`, so it holds `/data`
read-write for the same reason Radarr does.

**Then turn Plex's own subtitle agent back off.** Two things fetching subtitles
for the same file produce duplicates that are tedious to unpick, and Bazarr is
much better at it — more providers, scoring, per-language rules, and upgrades
over time.

## Plex (`http://<host>:32400/web`)

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
for audiobooks. Adding one is [apps/README.md](../apps/README.md), and it would mount `/mnt/active/books`
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
and joins the host's `render` group via `RENDER_GID` ([root §3](../README.md)); the remaining
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
`hw/x86.yaml`, or something passed `-f` ([root §3](../README.md)).
