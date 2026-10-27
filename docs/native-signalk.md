# Plan: run SignalK natively, keep the rest in Docker

Status: planned (2026-10-27). Not started.

## Why

SignalK is a hardware hub; every integration has needed a hole through the container: serial passthrough by-id, the BlueZ DBus mount, and (next) host networking for SocketCAN `can0`, which then breaks the `80:3000` mapping and collides with Grafana on 3000. mDNS discovery is also broken — multicast doesn't leave the bridge network, and the server would advertise its in-container port 3000 anyway — so SensESP devices and the espos BLE gateway can't find the server.

Native SignalK removes all of that. InfluxDB, Grafana and Jellyfin are plain network services and stay in Docker.

## Facts from the SignalK repo (checked 2026-10-27)

- `docs/installation/raspberry_pi_installation.md`: 64-bit Raspberry Pi OS (Pi 3/4/5), **Node.js 24** from NodeSource, `sudo npm install -g npm@latest`, then
  `sudo npm install -g signalk-server --allow-scripts=@canboat/canboatjs`
  (the `--allow-scripts` lets canboatjs build its native SocketCAN addon). mDNS deps: `libnss-mdns avahi-utils libavahi-compat-libdnssd-dev`. `package.json` engines: `node >= 22`.
- `bin/signalk-server-setup` writes `signalk.service` (runs as the invoking `SUDO_USER`, sets `EXTERNALPORT`) and `signalk.socket` (`ListenStream=<port>`). systemd owns port 80 and hands the socket to the unprivileged server (`LISTEN_FDS`, see `src/ports.ts`) — no root, no sysctl.
- mDNS: advertises `_http._tcp`, `_signalk-http._tcp`, `_signalk-ws._tcp` on `EXTERNALPORT` (`src/mdns.js`).

## Steps

1. **Check the OS is 64-bit** (`uname -m` → `aarch64`). If not, stop: Node 24 does not support 32-bit ARM.
2. **Back up**: `/mnt/storage/docker/signalk` (tarball) and `.env`, per `Recovery.md`.
3. **Install Node 24** (NodeSource) and npm; check `node -v` ≥ v24.
4. **Install SignalK pinned** to the version currently running in Docker (same config format):
   `sudo npm install -g signalk-server@<version> --allow-scripts=@canboat/canboatjs`
5. **Stop the Docker SignalK** (`docker compose stop signalk`) — keep its definition in `docker-compose.yml`, commented out, as the rollback.
6. **Config directory**: point SignalK at the existing `/mnt/storage/docker/signalk` (choose it in `signalk-server-setup`), so settings and installed plugins carry over. Fix ownership for the service user (container ran as `node`, uid **unverified** — check with `ls -ln`).
7. **Run `sudo signalk-server-setup`**: port 80, systemd yes, the config dir above.
8. **Plugins with native or external deps**: reinstall/rebuild where needed (`npm rebuild` in the config dir). `signalk-victron-ble` uses the Python `victron-ble` library — **check its README for how it finds Python on a native install** (unverified).
9. **Devices**: the serial ports are now used directly — set SignalK's connections to the stable names (`/dev/serial/by-id/...` for the Atom N2K gateway, GPS, Vesper AIS) or keep udev symlinks (`/dev/ttyNGT1`, `/dev/ttyGPS`, `/dev/ttyAIS`) via a `/etc/udev/rules.d/99-marine.rules`.
10. **InfluxDB**: the SignalK → InfluxDB plugin now reaches InfluxDB at `http://localhost:8086` (published by Docker).
11. **mDNS**: confirm with `avahi-browse -rt _signalk-http._tcp` from another machine — should show port 80.
12. **Verify**: admin UI, plugin status (no errors), Data Browser shows N2K/GPS/AIS/Victron/Ruuvi, Grafana still receives data, reboot survives.
13. **Remove** the commented SignalK service from `docker-compose.yml` after a week without issues; drop `SIGNALK_VERSION` from `.env`.
14. **Update `Recovery.md`**: native SignalK install + restore section; upgrade becomes `sudo npm install -g signalk-server@<new>` then `sudo systemctl restart signalk`.

## Rollback

`sudo systemctl disable --now signalk.socket signalk.service`, uncomment the Docker service, `docker compose up -d signalk`. Restore the config tarball if the native server changed it.

## After this

SocketCAN (`can0`, AtomS3 gs_usb gateway) needs nothing extra: a "Canbus (canboatjs)" connection on `can0`, with `can0` brought up by systemd-networkd (see the `atoms3-can-gateway` README).
