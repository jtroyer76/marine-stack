# Run SignalK natively, keep the rest in Docker

Status: done. SignalK runs on the host under systemd (socket activation on port 80); InfluxDB, Grafana and Jellyfin stay in Docker. The Docker `signalk` service is kept behind the `signalk-docker` compose profile as a rollback.

## Why

SignalK is a hardware hub; every integration needed a hole through the container: serial passthrough by-id, the BlueZ DBus mount, and (next) host networking for SocketCAN `can0`, which then breaks the `80:3000` mapping and collides with Grafana on 3000. mDNS discovery was also broken — multicast doesn't leave the bridge network, and the server advertised its in-container port 3000 anyway — so SensESP devices and BLE gateways couldn't find the server.

InfluxDB, Grafana and Jellyfin are plain network services and stay in Docker.

## Facts from the SignalK repo

- `docs/installation/raspberry_pi_installation.md`: 64-bit Raspberry Pi OS (Pi 3/4/5), Node.js 24 from NodeSource, then
  `sudo npm install -g signalk-server --allow-scripts=@canboat/canboatjs`
  (`--allow-scripts` lets canboatjs build its native SocketCAN addon). mDNS deps: `libnss-mdns avahi-utils libavahi-compat-libdnssd-dev`.
- `bin/signalk-server-setup` writes `signalk.service` (runs as `SUDO_USER`, sets `EXTERNALPORT`) and `signalk.socket` (`ListenStream=<port>`). systemd owns port 80 and hands the socket to the unprivileged server — no root, no sysctl.
- **Don't run `signalk-server-setup` on an existing config you want to keep as is.** Even with "Update" it rewrites `settings.json`, `package.json` and `baseDeltas.json` and asks for vessel name and MMSI again. Write the two unit files by hand instead (below).

## Migration

Same SignalK version and same Node major as the container, same config directory: settings, connections, plugins and security all carry over.

1. **Checks:** `uname -m` → `aarch64` (Node 24 has no 32-bit ARM). Note the container's versions: `docker exec signalk node -v` and the server version (server info at `http://<host>/signalk`). The container's `node` user is uid 1000; if your host user is too (`id`), no ownership changes are needed.
2. **Back up** the config: `cd /mnt/storage/docker && sudo tar czf ~/signalk-backup-$(date +%F).tgz signalk`
3. **Stable serial names on the host.** Inside the container the devices got their names from the compose `devices:` mapping; natively a udev rule gives the same names, so the SignalK connections don't change. Get each device's `ID_SERIAL` (or vendor/product id) with
   `for d in /dev/serial/by-id/*; do echo "== $d"; udevadm info -q property -n "$d" | grep -E 'ID_SERIAL=|ID_VENDOR_ID|ID_MODEL_ID'; done`
   and write `/etc/udev/rules.d/99-marine.rules`:
   ```
   SUBSYSTEM=="tty", ENV{ID_SERIAL}=="<n2k gateway ID_SERIAL>", SYMLINK+="ttyNGT1"
   SUBSYSTEM=="tty", ATTRS{idVendor}=="067b", ATTRS{idProduct}=="2303", SYMLINK+="ttyGPS"
   SUBSYSTEM=="tty", ENV{ID_SERIAL}=="<ais ID_SERIAL>", SYMLINK+="ttyAIS"
   ```
   Match by vendor/product id when a device's serial number isn't unique or stable (cheap Prolific adapters). Apply with `sudo udevadm control --reload-rules && sudo udevadm trigger --subsystem-match=tty`; check `ls -l /dev/ttyNGT1 /dev/ttyGPS /dev/ttyAIS`.
4. **mDNS deps:** `sudo apt-get install -y libnss-mdns avahi-utils libavahi-compat-libdnssd-dev`
5. **Node,** same major as the container, so plugins with native addons in the config's `node_modules` run as they are:
   `curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -` then `sudo apt-get install -y nodejs`
6. **SignalK, pinned to the container's version:**
   `sudo npm install -g signalk-server@<version> --allow-scripts=@canboat/canboatjs`
7. **Python plugins.** A plugin that builds a Python venv at install time built it inside the container, for the container's Python and paths; it won't run on the host. `signalk-victron-ble` does this (`preinstall` creates `ve/`; needs `python3-venv`). Record the exact package versions while the container still runs:
   `docker exec signalk /home/node/.signalk/node_modules/signalk-victron-ble/ve/bin/python -m pip freeze > ~/victron-freeze.txt`
8. **Stop the container** (downtime starts): `docker compose stop signalk`
9. **Rebuild the venv** for the host's Python with the same versions:
   ```bash
   cd /mnt/storage/docker/signalk/node_modules/signalk-victron-ble
   mv ve ve.docker
   python3 -m venv ve && ve/bin/python -m pip install wheel && ve/bin/python -m pip install -r ~/victron-freeze.txt
   ```
   If a pinned version won't install on the host's Python, use the plugin's own command instead: `ve/bin/python -m pip install -U -r requirements.txt`.
10. **Container hostnames → localhost.** The InfluxDB plugin reached InfluxDB by its container name; it now uses the published port:
    `sed -i 's#http://influxdb:8086#http://localhost:8086#' /mnt/storage/docker/signalk/plugin-config-data/signalk-to-influxdb2.json`
11. **systemd units** (what `signalk-server-setup` writes, plus waiting for the data mount). `/etc/systemd/system/signalk.service`:
    ```ini
    [Unit]
    Description=Signal K server
    RequiresMountsFor=/mnt/storage

    [Service]
    ExecStart=/usr/bin/signalk-server -c /mnt/storage/docker/signalk
    Restart=always
    WorkingDirectory=/mnt/storage/docker/signalk
    User=<your user>
    Environment=EXTERNALPORT=80
    Environment=NODE_ENV=production
    Environment=RUN_FROM_SYSTEMD=true

    [Install]
    WantedBy=multi-user.target
    ```
    `/etc/systemd/system/signalk.socket`:
    ```ini
    [Socket]
    ListenStream=80

    [Install]
    WantedBy=sockets.target
    ```
    Then `sudo systemctl daemon-reload && sudo systemctl enable signalk.service signalk.socket && sudo systemctl start signalk.socket signalk.service`. Logs: `journalctl -u signalk`.
12. **Verify:** server info at `http://<host>/signalk`, every source updating in the Data Browser, no plugin errors, Grafana receiving new points, `avahi-browse -rt _signalk-http._tcp` shows port 80. NMEA 0183 over TCP (port 10110) is served directly; the container published it with a port mapping. Reboot once to prove it comes back on its own.

## After the move

- **Source priorities.** Without priorities, whichever source reports first after a restart wins a path. A GPS with a week-rollover bug (dates in 2007) can win `navigation.datetime`, and plugins that use the date go wrong (tide predictions for the wrong year). Set priorities for `navigation.datetime` and `navigation.position` on the admin UI's Source Priorities page.
- **mDNS on Docker hosts.** avahi advertises on every Docker bridge and veth interface by default. Limit it to the real network in `/etc/avahi/avahi-daemon.conf` with `allow-interfaces=wlan0,eth0` (check the names with `ip -br link`), then restart `avahi-daemon` and `signalk`.
- **Audio for `signalk-notification-player`.** The container got its audio tools from `signalk/Dockerfile`; natively install the same: `sudo apt-get install -y alsa-utils mpg321 festival festvox-kallpc16k`. To play on a USB speaker rather than the Pi's headphone jack, set the default ALSA device for the service user in `~/.asoundrc` (the container mounted `signalk/asoundrc`, which picks the card by number). By card name survives renumbering (names from `aplay -l`):
  ```
  pcm.!default { type plug slave.pcm "hw:CARD=<card name>" }
  ctl.!default { type hw card <card name> }
  ```
- **Clean-up after a week without issues:** remove `ve.docker`, the backup tarball and the freeze file. `sudo apt autoremove` is safe: NodeSource's Node replaces Debian's `node-*` packages.

## Upgrade

`sudo npm install -g signalk-server@<new> --allow-scripts=@canboat/canboatjs`, then `sudo systemctl restart signalk`.

## Rollback

`sudo systemctl disable --now signalk.socket signalk.service`, put the InfluxDB URL back to `http://influxdb:8086`, restore the venv (`rm -rf ve && mv ve.docker ve`), then `docker compose --profile signalk-docker up -d --build signalk` (builds the image with the audio tools). Restore the config tarball if the native server changed anything else.

## SocketCAN

`can0` (gs_usb adapter) needs nothing extra natively: a "Canbus (canboatjs)" connection on `can0`, with `can0` brought up by systemd-networkd.
