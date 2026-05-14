# 🛡️ CrowdSec IPS Deployment for Raspberry Pi (ARM64)

## Project Overview
This repository contains the deployment configuration for an **Intrusion Prevention System (IPS)** using CrowdSec, optimized for home laboratory environments. This setup is designed for ARM64 architecture (Raspberry Pi) and provides "Active Defense" by monitoring system logs and automatically banning malicious actors via the host firewall.

### Architecture
*   **Detection Engine (Brain):** Containerized CrowdSec Local API (LAPI) monitoring SSH and system logs.
*   **Remediation (Muscle):** Host-based `iptables` firewall bouncer (`crowdsec-firewall-bouncer`).
*   **Cloud Intelligence:** Synced with the CrowdSec Console for real-time monitoring and community threat feeds.
*   **Communication:** Internal communication restricted to localhost (`http://127.0.0.1:8080/`).

---

## Prerequisites
*   Raspberry Pi (ARM64) running a Debian-based OS (e.g., Raspberry Pi OS, Ubuntu Server).
*   Docker and Docker Compose installed.
*   Standard system logging enabled (`/var/log/auth.log` and `/var/log/syslog`).

---

## Step-by-Step Installation Guide

### Step 1: Pre-Configure Host Log Files
*Crucial Step: Modern Debian systems may not create physical log files by default. If Docker attempts to mount a missing file, it will create a directory instead, crashing the parser*.

Create the empty log files and set permissions on the host:
```bash
sudo touch /var/log/auth.log /var/log/syslog
sudo chmod 644 /var/log/auth.log /var/log/syslog
```

### Step 2: Directory and Volume Setup
Create the project directory and ensure proper ownership for your user account:
```bash
mkdir -p ~/crowdsec-ips/config
mkdir -p ~/crowdsec-ips/data
cd ~/crowdsec-ips
sudo chown -R $USER:$USER ~/crowdsec-ips/config
```

### Step 3: Configure Docker Compose
Create `docker-compose.yml` with ARM-optimized configuration:
```bash
nano docker-compose.yml
```
Paste the following:
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
      - /var/log/auth.log:/var/log/auth.log:ro
      - /var/log/syslog:/var/log/syslog:ro
      - ./config:/etc/crowdsec
      - ./data:/var/lib/crowdsec/data
    ports:
      - "8080:8080"
```

### Step 4: Configure Acquisition (`acquis.yaml`)
Define the logs the engine must monitor. This file must be placed in `~/crowdsec-ips/config/acquis.yaml`:
```yaml
filenames:
  - /var/log/auth.log
  - /var/log/syslog
labels:
  type: syslog
```

### Step 5: Launch & Initialize
Start the stack and install core security collections:
```bash
docker compose up -d
docker exec crowdsec cscli hub update
docker exec crowdsec cscli collections install crowdsecurity/linux crowdsecurity/sshd
docker exec crowdsec cscli reload
```

### Step 6: Enroll in Cloud Console
To resolve dashboard sync lag and enable remote monitoring, enroll the engine using your console key:
```bash
docker exec crowdsec cscli console enroll <YOUR_ENROLL_KEY>
```

### Step 7: Install Host Firewall Bouncer
Install the bouncer on the host OS to interface with `iptables`:
1. Add repository: `curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash`
2. Install: `sudo apt update && sudo apt install crowdsec-firewall-bouncer -y`
3. Generate LAPI key: `docker exec crowdsec cscli bouncers add host-bouncer`
4. Update config at `/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml`:
   * `api_url: http://127.0.0.1:8080/`
   * `api_key: <YOUR_GENERATED_KEY>`
5. Restart: `sudo systemctl restart crowdsec-firewall-bouncer`

---

## 🔍 Verification & Testing

### 1. Check Metrics
Verify the engine is successfully reading logs:
*   **Command:** `docker exec crowdsec cscli metrics`
*   **Requirement:** `file:/var/log/auth.log` must show **Lines read > 0**.

### 2. Verify Connectivity
*   **Bouncer:** `docker exec crowdsec cscli bouncers list` (Status should be **Valid**).
*   **Cloud:** `docker exec crowdsec cscli console status` (All options should be **Green**).

### 3. Simulation Attack
Validate the IPS by simulating a brute-force attack. *Note: Using 8.8.8.8 will fail as it is whitelisted by default*.
```bash
for i in {1..50}; do echo "$(date '+%b %d %H:%M:%S') $(hostname) sshd[$(($$))]: Failed password for invalid user bot-test from 1.2.3.4 port 5678 ssh2" | sudo tee -a /var/log/auth.log; done
```
**Check for Ban:** `docker exec crowdsec cscli decisions list` should list **1.2.3.4** with origin **lapi**.

---

## Useful Commands
*   **Manual Ban:** `docker exec crowdsec cscli decisions add -i <IP>`
*   **Unban IP:** `docker exec crowdsec cscli decisions delete -i <IP>`
*   **View Engine Logs:** `docker logs crowdsec`
````</YOUR_ENROLL_KEY>
