# home — Home Assistant, Mosquitto and Zigbee2MQTT

The home-automation stack, defined in [`compose.yaml`](compose.yaml) and
assembled by the root [`compose.yaml`](../compose.yaml). Read
[the root README](../README.md) first for the disks, `.env` and the first-run
order.

| Service | LAN address | Notes |
| --- | --- | --- |
| Home Assistant | `http://<host>:8123` | `network_mode: host`, so no `ports:` entry |
| Mosquitto (MQTT) | `<host>:1883` | published so host-networked HA can reach it |
| Mosquitto (websockets) | `<host>:9001` | |
| Zigbee2MQTT | `http://<host>:8082` | deCONZ wants the same port — see below |

---

## Home Assistant, Mosquitto and Zigbee2MQTT

`http://<host>:8123`, `http://<host>:8082` for the Zigbee frontend. Mosquitto has
no UI. All three keep their state under `/srv/appdata/home/`. Home Assistant
onboards through its own UI on first visit; Mosquitto needs the config file
below *before* it will accept a single connection.

**Two conflicts with what was already running.** Neither is a port clash; the
port registry ([root §4](../README.md)) is clean. Both come from Home Assistant needing the host's own
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
`/srv/appdata/home/mosquitto/config/mosquitto.conf` is required rather than
optional. A broker that accepts no connections looks exactly like a broker that
is down, and nothing in the log says "you forgot a file".

Write it before the first start:

```bash
sudo mkdir -p /srv/appdata/home/mosquitto/{config,data}
sudo tee /srv/appdata/home/mosquitto/config/mosquitto.conf >/dev/null <<'EOF'
listener 1883
protocol mqtt

listener 9001
protocol websockets

allow_anonymous true
persistence true
persistence_location /mosquitto/data/
log_dest stdout
EOF
# the eclipse-mosquitto image runs as uid 1883, NOT 1000 like everything else
sudo chown -R 1883:1883 /srv/appdata/home/mosquitto
```

**That last line is the one that catches people.** Every other directory under
`/srv/appdata` is owned by `1000:1000`, and `bin/bootstrap` sets it that way
wholesale — but this image runs as uid 1883 and cannot write its persistence
database or log under 1000. The symptom is a broker that starts, accepts
connections, and loses every retained message on restart.

`allow_anonymous true` is deliberate and matches the LAN-only posture: the
broker is reachable only from this network. If you would rather not, swap it for
`password_file /mosquitto/config/passwd` and generate the file with
`docker compose exec mosquitto mosquitto_passwd -c /mosquitto/config/passwd <user>`,
then set the same credentials in Home Assistant and Zigbee2MQTT.

**Who talks to whom**, since the three sit on different networks:

| From | To | Address | Why |
| --- | --- | --- | --- |
| Zigbee2MQTT | Mosquitto | `mqtt://mosquitto:1883` | both ordinary bridge services |
| Home Assistant | Mosquitto | `localhost:1883` | HA is host-networked, so it uses the *published* port, not the service name |

That second row is the one that looks wrong and is not: a host-networked
container is not on the compose network, so `mosquitto` does not resolve from it
— the same rule as [the gluetun namespace rule](../media/README.md#the-rule-that-breaks-things-first), from the same direction as Jellyseerr reaching Plex ([apps/README.md](../apps/README.md)).

**Not on the tunnel, deliberately.** Home Assistant controls your house and runs
privileged; [root §14](../README.md) covers why it gets no hostname. If you do want it remotely, it is
[root §9](../README.md) plus an Access policy, and worth more thought than the media apps needed.
