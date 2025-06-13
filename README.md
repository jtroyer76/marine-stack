# Marine Stack: SignalK + Grafana + InfluxDB

A self-hosted marine data system using Docker Compose. This stack includes SignalK for marine/NMEA data collection, InfluxDB for time-series storage, and Grafana for real-time dashboards. Designed for use on boats, Raspberry Pi, NUCs, or other Linux servers.

---

## 📦 Included Services

| Service   | Port | Description                          |
|-----------|------|--------------------------------------|
| SignalK   | 80   | Marine data server and API platform  |
| Grafana   | 3000 | Dashboard and data visualization     |
| InfluxDB  | 8086 | Time-series database backend         |

---

## 🚀 Quick Start

Follow these steps to deploy the stack:

### 1. Install Docker and Docker Compose

**Debian/Ubuntu**

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian   $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Test installation:

```bash
docker version
docker compose version
```

---

### 2. Clone the Repository

```bash
git clone https://github.com/jtroyer76/marine-stack.git
cd marine-stack
```

---

### 3. Create Data Directories

```bash
mkdir -p ~/data/signalk ~/data/grafana ~/data/influx
```

---

### 4. Launch the Stack

```bash
docker compose up -d
```

Access services:

- SignalK: `http://<your-host-ip>`
- Grafana: `http://<your-host-ip>:3000`
- InfluxDB: `http://<your-host-ip>:8086`

---

## 🔐 Default Credentials

| Service  | Username | Password   |
|----------|----------|------------|
| Grafana  | admin    | changeme   |
| InfluxDB | admin    | changeme   |
| SignalK  | Set on first launch |

---

## 🗂 Data Volumes

| Container | Host Path         | Container Path                |
|-----------|-------------------|-------------------------------|
| SignalK   | ~/data/signalk    | /home/node/.signalk           |
| Grafana   | ~/data/grafana    | /var/lib/grafana              |
| InfluxDB  | ~/data/influx     | /var/lib/influxdb             |

---

## 🔁 Restarting

```bash
docker compose down
docker compose up -d
```

---

## 💾 Backup & Restore

To backup:

```bash
tar -czf marine-backup.tar.gz ~/data
```

To restore:

```bash
tar -xzf marine-backup.tar.gz -C ~/
docker compose up -d
```

---

## 📜 License

MIT License — free to use, modify, and distribute.
