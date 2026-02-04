# Neptune Recovery Guide

If your boot SSD dies or you need to replace the Pi, follow these steps. Your data is safe on the RAID array.

## What You Have

- Raspberry Pi 4 in Argon M.2 case
- 128GB Transcend M.2 boot SSD (sdc)
- 2x 2TB Crucial BX500 in RAID1 (md127)
- Docker stack: SignalK, InfluxDB, Grafana

## 1. Flash Raspberry Pi OS

Use Raspberry Pi Imager:

- OS: Raspberry Pi OS Lite (64-bit)
- Hostname: `neptune`
- Username: `jess`
- Enable SSH
- Configure WiFi (backup, ethernet preferred)

Boot from the Argon M.2 SSD.

## 2. Initial Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git mdadm samba -y
```

## 3. Mount the RAID

The array should auto-detect. Check:

```bash
cat /proc/mdstat
```

If it shows `md127` with `[UU]`, it's healthy. If not assembled:

```bash
sudo mdadm --assemble --scan
```

Create mount point and add to fstab:

```bash
sudo mkdir -p /mnt/storage
```

Get UUID:

```bash
sudo blkid /dev/md127
```

Edit fstab:

```bash
sudo nano /etc/fstab
```

Add:

```
UUID=74bb4190-779b-4827-bba1-82cfb466f6a5  /mnt/storage  ext4  defaults,nofail  0  2
```

Mount it:

```bash
sudo mount -a
df -h /mnt/storage
```

Save RAID config:

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

## 4. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

Log out and back in, then verify:

```bash
docker --version
```

## 5. Clone Repo and Configure

```bash
mkdir -p ~/src
cd ~/src
git clone https://github.com/jtroyer76/marine-stack.git
cd marine-stack
```

Restore your .env from backup:

```bash
cp /mnt/storage/backup/marine-stack.env .env
```

Or create new one if backup is missing:

```bash
nano .env
```

```env
MARINE_DATA=/mnt/storage/docker

SIGNALK_VERSION=v2.16.0
INFLUXDB_VERSION=2.7
GRAFANA_VERSION=11.2.0

INFLUXDB_ADMIN_USER=admin
INFLUXDB_ADMIN_PASSWORD=<your_password>
INFLUXDB_ADMIN_TOKEN=<your_token>
INFLUXDB_ORG=boat
INFLUXDB_BUCKET=signalk

GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=<your_password>
```

## 6. Start the Stack

```bash
docker compose up -d
docker ps
```

All your data (SignalK config, InfluxDB history, Grafana dashboards) is on the RAID and will be there.

## 7. Configure Samba

```bash
sudo nano /etc/samba/smb.conf
```

Replace contents with:

```ini
[global]
    workgroup = WORKGROUP
    server string = Neptune
    security = user
    map to guest = Bad User
    guest account = nobody

[media]
    path = /mnt/storage/media
    browseable = yes
    read only = yes
    guest ok = yes
    write list = jess
```

Set Samba password and restart:

```bash
sudo smbpasswd -a jess
sudo systemctl restart smbd
```

## 8. Verify Everything

- SignalK: http://neptune:80
- InfluxDB: http://neptune:8086
- Grafana: http://neptune:3000
- Samba: smb://neptune/media

---

## Useful Commands

### RAID Status

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md127
```

### Docker

```bash
# Status
docker ps

# Logs
docker logs signalk
docker logs influxdb
docker logs grafana

# Upgrade one container
docker compose pull signalk
docker compose up -d signalk

# Restart everything
docker compose down
docker compose up -d
```

### RAID Check Control

```bash
# Pause check (for faster file transfers)
sudo sh -c 'echo idle > /sys/block/md127/md/sync_action'

# Resume check
sudo sh -c 'echo check > /sys/block/md127/md/sync_action'

# Check speed
cat /proc/mdstat
```

### Disk Usage

```bash
df -h /mnt/storage
du -sh /mnt/storage/*
```

---

## Backup Locations

- `.env`: `/mnt/storage/backup/marine-stack.env`
- SignalK config: `/mnt/storage/docker/signalk`
- InfluxDB data: `/mnt/storage/docker/influx`
- Grafana dashboards: `/mnt/storage/docker/grafana`
- Media: `/mnt/storage/media`

---

## Monthly Maintenance

RAID scrub runs automatically on the 1st at 3am (if cron configured):

```bash
sudo crontab -e
```

Add:

```
0 3 1 * * /usr/sbin/mdadm --action=check /dev/md127
```

---

## If a Drive Fails

Check which drive:

```bash
sudo mdadm --detail /dev/md127
```

Replace the failed drive, then add to array:

```bash
sudo mdadm --add /dev/md127 /dev/sdX
```

Monitor rebuild:

```bash
watch cat /proc/mdstat
```
