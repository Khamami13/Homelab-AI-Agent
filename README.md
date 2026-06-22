# Homelab AI Agent Server Setup Guide

### Debian + Btrfs + Docker + Portainer + 9Router + Hermes Agent + n8n

Author: Rizqi Khamami

---

# 1. Spesifikasi Server

Hardware:

* Dell OptiPlex 3070
* Intel Core i5
* RAM 8GB
* NVMe 256GB
* Debian Server
* Docker Engine
* Btrfs Snapshot

Target Service:

* Portainer
* 9Router
* Hermes Agent
* n8n
* MikroTik Automation
* Cisco Automation
* OLT Automation
* Product Scraping
* AI Agent

---

# 2. Instalasi Debian

Saat instalasi Debian:

Filesystem:

```text
EFI        1 GB      FAT32
ROOT       Sisa Disk Btrfs
```

Mount Point:

```text
/boot/efi
/
```

Pilih:

```text
BTRFS
```

Karena mendukung:

* Snapshot
* Compression
* Rollback
* Copy-on-write

---

# 3. Update Sistem

Login:

```bash
ssh khamami@SERVER-IP
```

Update:

```bash
sudo apt update
sudo apt upgrade -y
```

Install tools dasar:

```bash
sudo apt install -y \
curl \
wget \
git \
nano \
vim \
htop \
btop \
fastfetch \
ca-certificates \
gnupg \
lsb-release \
dnsutils
```

---

# 4. Konfigurasi Sudo

Masuk root:

```bash
su -
```

Tambahkan user:

```bash
usermod -aG sudo khamami
```

Logout:

```bash
exit
```

Login ulang.

Cek:

```bash
groups
```

Harus muncul:

```text
sudo
```

---

# 5. Snapshot Btrfs

Cek:

```bash
findmnt /
```

Contoh:

```text
btrfs
```

Buat folder snapshot:

```bash
sudo mkdir -p /.snapshots
```

Snapshot instalasi bersih:

```bash
sudo btrfs subvolume snapshot -r / \
/.snapshots/debian-fresh-install-$(date +%Y%m%d-%H%M)
```

Cek:

```bash
sudo btrfs subvolume list /
```

---

# 6. Install Docker

Hapus docker lama:

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

Install:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Tambahkan user:

```bash
sudo usermod -aG docker khamami
```

Logout lalu login kembali.

Tes:

```bash
docker version
```

---

# 7. Install Docker Compose

Cek:

```bash
docker compose version
```

Compose Plugin sudah tersedia pada Docker terbaru.

---

# 8. Struktur Direktori

```text
/opt/junkielab
├── portainer
├── 9router
├── hermes-agent
└── n8n
```

Buat:

```bash
sudo mkdir -p /opt/junkielab
sudo chown -R khamami:khamami /opt/junkielab
```

---

# 9. Install Portainer

```bash
mkdir -p /opt/junkielab/portainer
cd /opt/junkielab/portainer
```

docker-compose.yml

```yaml
services:
  portainer:
    image: portainer/portainer-ce:2.42.0
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

Jalankan:

```bash
docker compose up -d
```

Akses:

```text
https://SERVER-IP:9443
```

---

# 10. Install Hermes Agent

Buat folder:

```bash
mkdir -p /opt/junkielab/hermes-agent
cd /opt/junkielab/hermes-agent
```

---

# 11. Dockerfile Hermes NetOps

Buat:

```bash
nano Dockerfile
```

Isi:

```dockerfile
FROM nousresearch/hermes-agent:latest

USER root

ENV DEBIAN_FRONTEND=noninteractive
ENV VIRTUAL_ENV=/opt/netops-venv
ENV PATH="/opt/netops-venv/bin:$PATH"

RUN apt-get update && apt-get install -y --no-install-recommends \
    openssh-client \
    sshpass \
    curl \
    wget \
    dnsutils \
    iputils-ping \
    netcat-openbsd \
    traceroute \
    mtr-tiny \
    jq \
    ca-certificates \
    git \
    nano \
    vim-tiny \
    python3 \
    python3-pip \
    python3-venv \
    python3-dev \
    build-essential \
    libssl-dev \
    libffi-dev \
    iproute2 \
    net-tools \
    telnet \
    whois \
    snmp \
    snmpd \
    snmptrapd \
    tftp-hpa \
    lftp \
    expect \
    tcpdump \
    httpie \
    chromium \
    chromium-driver \
    fonts-noto \
    fonts-noto-color-emoji \
    fonts-liberation \
    && rm -rf /var/lib/apt/lists/*

RUN python3 -m venv /opt/netops-venv && \
    pip install --upgrade pip setuptools wheel && \
    pip install \
    paramiko \
    netmiko \
    napalm \
    scrapli \
    ncclient \
    pysnmp \
    textfsm \
    ntc-templates \
    jinja2 \
    pyyaml \
    requests \
    httpx \
    routeros-api \
    librouteros \
    rich \
    typer \
    beautifulsoup4 \
    pandas \
    openpyxl \
    playwright \
    scrapy \
    lxml \
    selectolax \
    selenium \
    advertools \
    tldextract \
    rapidfuzz \
    dateparser

WORKDIR /opt/data

USER root
```

---

# 12. Docker Compose Hermes

Buat:

```bash
nano docker-compose.yml
```

Isi:

```yaml
services:
  hermes-agent:
    build:
      context: .
      dockerfile: Dockerfile

    image: junkielab/hermes-agent-netops:latest

    container_name: hermes-agent

    restart: unless-stopped

    network_mode: host

    command: gateway run

    dns:
      - 1.1.1.1
      - 8.8.8.8
      - 9.9.9.9

    environment:
      TZ: Asia/Jakarta
      HERMES_DASHBOARD: "0"

    volumes:
      - /opt/junkielab/hermes-agent/data:/opt/data
      - /var/run/docker.sock:/var/run/docker.sock

    cap_add:
      - NET_RAW
```

Build:

```bash
docker compose build --no-cache
```

Run:

```bash
docker compose up -d
```

---

# 13. Install n8n

```bash
mkdir -p /opt/junkielab/n8n
cd /opt/junkielab/n8n
```

Buat:

```bash
mkdir n8n_data
mkdir postgres_data
```

docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:17
    restart: unless-stopped
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: StrongPassword
    volumes:
      - ./postgres_data:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: StrongPassword
    volumes:
      - ./n8n_data:/home/node/.n8n
```

Jalankan:

```bash
docker compose up -d
```

Akses:

```text
http://SERVER-IP:5678
```

---

# 14. Backup

Backup seluruh stack:

```bash
sudo tar -czf \
junkielab-backup-$(date +%Y%m%d).tar.gz \
/opt/junkielab
```

---

# 15. Update Semua Container

```bash
docker compose pull
docker compose up -d
```

Atau melalui Portainer.

---

# 16. Verifikasi Hermes

Masuk container:

```bash
docker exec -it hermes-agent bash
```

Test:

```bash
which ssh
which sshpass
which dig
which snmpwalk
which chromium
```

Python:

```bash
python -c "import netmiko; print('OK')"
python -c "import pandas; print('OK')"
python -c "import playwright; print('OK')"
```

---

# 17. Kemampuan Hermes

Network Automation:

* MikroTik
* Cisco
* Huawei
* Juniper
* OLT ZTE
* OLT Huawei
* OLT FiberHome
* SNMP Monitoring

Marketing Automation:

* Product Scraping
* Price Monitoring
* SEO Crawling
* Excel Export
* Browser Automation

AI Automation:

* n8n Integration
* Webhook
* API Integration
* Telegram Bot
* WhatsApp Automation
* Dashboard Monitoring

---

# 18. Snapshot Setelah Semua Berhasil

```bash
sudo btrfs subvolume snapshot -r / \
/.snapshots/junkielab-production-$(date +%Y%m%d-%H%M)
```

Rollback dapat dilakukan jika terjadi kerusakan sistem.
