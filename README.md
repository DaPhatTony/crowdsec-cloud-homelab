# CrowdSec IPS Deployment for Raspberry Pi (ARM64)

## Project Overview
This repository contains the deployment configuration for an **Intrusion Prevention System (IPS)** using CrowdSec. It is optimized for ARM64 architecture (specifically Raspberry Pi) and designed to provide "Active Defense" by monitoring system logs and automatically banning malicious actors via the host firewall.

### Architecture
* **Detection Engine (Brain):** Containerized CrowdSec Local API (LAPI) monitoring SSH and system logs.
* **Remediation (Muscle):** Host-based `iptables` firewall bouncer (`crowdsec-firewall-bouncer`).
* **Communication:** Internal communication between the bouncer and LAPI is restricted to localhost (`http://127.0.0.1:8080/`).

---

## Prerequisites
* Raspberry Pi running a Debian-based OS (e.g., Raspberry Pi OS Bookworm, Ubuntu Server).
* Docker and Docker Compose installed.
* Standard system logging enabled (`/var/log/auth.log` and `/var/log/syslog`).

---

## Step-by-Step Installation Guide

### Step 1: Pre-Configure Host Log Files
*Crucial Step: Modern Debian systems often use `journald` and may not have physical log files created by default. If Docker attempts to mount a missing file, it will create a directory instead, which crashes the CrowdSec parser.*

Create the empty log files and set permissions on the host:
```bash
sudo touch /var/log/auth.log /var/log/syslog
sudo chmod 644 /var/log/auth.log /var/log/syslog
```

### Step 2: Configure Docker Compose
Create your project directory and a `docker-compose.yml` file:
```bash
mkdir ~/crowdsec-lab
cd ~/crowdsec-lab
nano docker-compose.yml
```

Paste the following ARM-optimized configuration. *(Note: We omit the auto-install `COLLECTIONS` environment variable to prevent startup crashes caused by Docker Hub timeouts).*

```yaml
services:
  crowdsec:
    image: crowdsecurity/crowdsec:latest
    platform: linux/arm64/v8
    container_name: crowdsec
    restart: always
    environment:
      - GID=1000
    volumes:
      - ./config:/etc/crowdsec
      - ./data:/var/lib/crowdsec/data
      - /var/log/auth.log:/var/log/auth.log:ro
      - /var/log/syslog:/var/log/syslog:ro
    ports:
      - "8080:8080" # Local API port
```

### Step 3: Launch & Initialize the Detection Engine
Start the container:
```bash
docker compose up -d
```

Once the container is running, manually update the hub and install the core security collections. This ensures stability during the initial boot phase:
```bash
docker exec crowdsec cscli hub update
docker exec crowdsec cscli collections install crowdsecurity/linux crowdsecurity/sshd
docker exec crowdsec cscli reload
```

### Step 4: Install the Firewall Bouncer (Host OS)
To allow CrowdSec to actually block traffic, we must install the Bouncer directly on the host operating system so it can interface with `iptables`.

1. Add the CrowdSec repository to your host system:
```bash
curl -s [https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh](https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh) | sudo bash
```

2. Install the bouncer package:
```bash
sudo apt update
sudo apt install crowdsec-firewall-bouncer -y
```
*Note: If the service fails to auto-start upon installation, run `sudo systemctl daemon-reload`.*

### Step 5: Connect the Bouncer to the Container
The host bouncer needs an API key to securely authenticate with the containerized Local API (LAPI).

1. Generate the API key inside the container:
```bash
docker exec crowdsec cscli bouncers add host-bouncer
```
**(Save the outputted key!)**

2. Edit the bouncer configuration on the host:
```bash
sudo nano /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```

3. Update the following fields in the YAML file:
* `api_url`: Ensure this is set to `http://127.0.0.1:8080/`
* `api_key`: Paste the key you generated in Step 5.1.

4. Save the file and restart the bouncer service:
```bash
sudo systemctl restart crowdsec-firewall-bouncer
```

### Step 6: Verification
Verify that the host bouncer has successfully connected to the containerized LAPI:
```bash
docker exec crowdsec cscli bouncers list
```
You should see `host-bouncer` listed with a status of **Valid**. Your Intrusion Prevention System is now fully active!

---

## Useful Commands
* **View Active Bans:** `docker exec crowdsec cscli decisions list`
* **Manually Ban an IP:** `docker exec crowdsec cscli decisions add -i <IP_ADDRESS>`
* **View Parsing Metrics:** `docker exec crowdsec cscli metrics`
