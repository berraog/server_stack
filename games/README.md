# games — ES-DE and RetroArch on the TV

A retro console on the same box: ES-DE on the HDMI output, driven by an Xbox pad
over Bluetooth. Defined in [`compose.yaml`](compose.yaml), which is short and
commented, and assembled by the root [`compose.yaml`](../compose.yaml).

Unlike every other stack here this one has **no port and no hostname** — the
output is the TV and the input is a controller. Read
[the root README](../README.md) first for the disks, `.env` and the first-run
order.

| Helper | What it does |
| --- | --- |
| [`bin/roms-import`](../bin/roms-import) | hardlinks downloaded ROMs into the tree ES-DE expects |
| [`bin/games-link-emulators`](../bin/games-link-emulators) | lets ES-DE find RetroArch across containers |

---

A retro console on the same box, on the HDMI output, driven by an Xbox pad over
Bluetooth. [`compose.yaml`](compose.yaml) is short and commented;
this section is the part that is not visible in the file.

## The two things that surprise people

**ES-DE does not emulate anything.** It is a frontend: a themed, controller-
driven menu that browses a library and shells out to an emulator. Ship it on its
own and you get a beautiful interface where every game fails to launch. The
emulators are the `retroarch` service, in its own container, and the shim in
this file.4 is what connects them.

**There is no display server to install.** The obvious design is a minimal Xorg
plus openbox on the host, with the container drawing on it. That works, but it
is strictly more to own: X packages on a server that otherwise has none, a
session to start at boot, a user to run it as, and `xhost` rules so the
container may connect. The image can run **its own Xorg** on the HDMI output
instead (`ESDE_USE_INTERNAL_X=1`), which is what this stack does. The host stays
a headless server. Nothing about X is installed on it, and the entire display
stack is deleted by `docker compose down`.

That choice has one consequence worth writing down. RetroArch is a *second*
container that must draw on the *same* screen, so ES-DE's X socket is shared out
through the `x11-socket` volume — read-write on `es-de`, which is the server
creating the socket, read-only on `retroarch`, which is only a client. Swapping
those gives a black screen and no useful error.

## Storage

| Path | Holds | Disk |
| --- | --- | --- |
| `/srv/appdata/games/es-de` | settings, themes, gamelists, scraped artwork | internal SSD |
| `/srv/appdata/games/retroarch` | cores, save files, save states | internal SSD |
| `/mnt/active/games/roms` | the library ES-DE reads — read-only to both containers | active USB |
| `/mnt/active/games/bios` | BIOS images for the cores that need them | active USB |

Save files live under `/srv/appdata`, like every other container's state, so
they are in the same backup as everything else. The ROM tree is bulk data and
belongs on `/mnt/active` with the rest of it.

## Getting ROMs in — `bin/roms-import`

**ES-DE does not scan for games.** It looks for one directory per system, named
exactly what ES-DE calls that system — `snes`, `megadrive`, `psx`, `n64` — and
ignores everything else on disk. A folder of mixed downloads produces an empty
frontend with no error, which is the most common way this looks broken when it
is not.

[`bin/roms-import`](../bin/roms-import) builds that tree out of whatever shape the
downloads arrived in, **by hardlink** — the same trick, for the same reason, as
`downloads/` and `library/` in [root §1](../README.md). One set of blocks, two names: the download
keeps the layout it came with, ES-DE sees the rigid tree it insists on, and the
bytes exist once. `df` counts them once.

```bash
bin/roms-import                 # dry run — prints the plan, changes nothing
bin/roms-import --apply         # create the links
bin/roms-import --apply ~/dump  # from somewhere other than /mnt/active/other
```

It resolves a file two ways, in this order:

1. **By extension**, for extensions that identify exactly one system — `.sfc`,
   `.nes`, `.z64`, `.gba`. The table is checked against ES-DE's own
   `es_systems.xml` and is deliberately incomplete: `.zip`, `.iso`, `.bin`,
   `.cue` and `.chd` are each claimed by dozens of systems, so they are not in
   it. A wrong guess files a game where nobody looks again.
2. **By folder name**, which is how disc-based systems get sorted at all. A
   folder called `PS1`, `PlayStation` or `Sega Saturn` is obvious to a person
   and invisible to the extension table. Spelling and punctuation do not
   matter — `Sega Mega-Drive`, `sega_megadrive` and `megadrive` all land in
   `megadrive`.

Anything it cannot resolve is **listed, not guessed**. Put those in a folder
named after the system and re-run.

Per-game folders are preserved only *below* a system folder, because a `.cue` is
useless without the `.bin` beside it:

```
/mnt/active/other/                      /mnt/active/games/roms/
  Super Nintendo/Mario.sfc      ->        snes/Mario.sfc
  misc/Metroid.nes              ->        nes/Metroid.nes
  PS1/Final Fantasy VII/*.cue   ->        psx/Final Fantasy VII/*.cue
  ROMs/Sega/Saturn/Panzer/*     ->        saturn/Panzer/*
```

Re-running is the normal way to pick up new downloads: a link that already
points at the right inode is skipped, and a *different* file with the same name
is reported and left alone, never overwritten. Restart ES-DE afterwards — it
reads its game list once, at startup:

```bash
cd ~/stack && docker compose restart es-de
```

`megadrive` or `genesis`, `snes` or `snesna` — ES-DE accepts both spellings of
the regional twins. The script picks the international ones; the table at the
top of it is the only place that decides, and nothing else depends on the choice.

## Bluetooth — pair on the host, not in the container

BlueZ owns the adapter, and the container gets the *result*: a paired pad shows
up as `/dev/input/event*`, which is already passed through. So pairing is a host
job, done once.

```bash
sudo apt install -y bluez
sudo systemctl enable --now bluetooth
bluetoothctl
```

Hold the Xbox pad's **pair** button (the small one on top, next to the bumper)
until the Xbox button flashes rapidly, then, inside `bluetoothctl`:

```
power on
agent on
default-agent
scan on                      # wait for "Xbox Wireless Controller"
pair    <MAC>
trust   <MAC>                # without this it will not reconnect after a reboot
connect <MAC>
scan off
exit
```

`trust` is the line people miss. Pairing without it works until the first
reboot, and then the pad connects to nothing and the TV shows a frozen menu.

Confirm the kernel sees it, and that it is inside the container too:

```bash
ls -l /dev/input/by-id/ | grep -i xbox
docker compose exec es-de sh -c 'ls /dev/input/event*'
```

Xbox pads are handled by the kernel's `xpad` driver over Bluetooth on modern
Ubuntu, so nothing extra is needed. If the pad connects but nothing moves in the
menu, it is an SDL mapping problem rather than a Bluetooth one — set
`SDL_GAMECONTROLLERCONFIG` on the `es-de` service.

## Connecting ES-DE to the emulators — `bin/games-link-emulators`

ES-DE launches a game by running `retroarch` and expects it on its own PATH.
RetroArch is in another container, so what goes on that PATH is RetroStack's
shim: it writes the command line into a FIFO the two containers share, waits,
and returns the exit code. The game runs in the `retroarch` container, on ES-DE's
display.

Nothing in compose can put a file inside another image, so this one step is a
script:

```bash
bin/games-link-emulators
```

**Re-run it whenever the `es-de` container is recreated** — a `pull`, an image
change or `up -d --force-recreate` all reset the container filesystem and take
the shim with it. Re-running at any other time is harmless. The symptom of
forgetting is every game failing instantly, or flashing back to the menu with no
message at all.

## First start

```bash
cd ~/stack
docker compose config -q            # catches a broken fragment before anything runs
docker compose up -d es-de retroarch
bin/games-link-emulators
```

The TV should show ES-DE within a few seconds. Then, in ES-DE: set the ROM
directory to `/roms`, and scrape artwork from the menu — it takes hours on a
large library, and lives in `/srv/appdata/games/es-de`, so it survives
everything except deleting that directory.

## When it does not work

| Symptom | Cause |
| --- | --- |
| TV stays on the console login | Xorg could not take the VT. `sudo systemctl mask getty@tty1` and restart `es-de`. |
| ES-DE starts, no games | The ROM tree is not laid out per system. `bin/roms-import` (this file.2), then restart `es-de`. |
| Games listed, every launch fails | The shim is missing. `bin/games-link-emulators` (this file.4). |
| Launch hangs forever, no error | The `emulator-control` volume did not mount, so the FIFO is not there. `docker compose exec es-de ls /run/retrostack-emulators` — if it is empty or missing, the tmpfs on `/run` shadowed it; `docker compose up -d --force-recreate es-de retroarch`. |
| Black screen after launching a game | The X socket is shared the wrong way round — `es-de` needs `x11-socket` read-**write**, `retroarch` read-only. |
| No sound | `ESDE_AUDIO_OUTPUT=hdmi` in `.env`. `auto` prefers USB and 3.5mm over HDMI. |
| Pad works, then not after a reboot | `trust <MAC>` was never run (this file.3). |

Two notes on living with this. ES-DE is the only **privileged** container on the
box — driving KMS, VTs and evdev needs it — which is exactly why it publishes no
port and is not exposed through the tunnel ([root §14](../README.md)); it is reachable only from the
sofa. And it holds the console: while it is running, tty1 belongs to Xorg, so
administer the box over SSH as usual and use `docker compose stop es-de` if you
need the physical console back.
