# Self-Hosted Vaultwarden with HTTPS (Docker & WSL Ubuntu)

A simplified, step-by-step guide to running **Vaultwarden** locally using Docker Compose, securing it with local SSL certificates via `mkcert` on WSL Ubuntu.

---

## 📋 Prerequisites

- **WSL (Ubuntu)** installed on Windows
- **Docker** and **Docker Compose** installed inside WSL or Docker Desktop integrated with WSL

---

## 🚀 Quick Setup Guide

### Step 1: Install `mkcert` for Local SSL

Install `mkcert` inside your WSL Ubuntu terminal to generate trusted self-signed certificates:

```bash
sudo apt update && sudo apt install -y libnss3-tools
curl -JLO "[https://dl.filippo.io/mkcert/latest?for=linux/amd64](https://dl.filippo.io/mkcert/latest?for=linux/amd64)"
chmod +x mkcert-v*-linux-amd64
sudo mv mkcert-v*-linux-amd64 /usr/local/bin/mkcert
```

### Step 2: Set Up Directory & Certificates
1. Create the project directory structure:

```bash
mkdir -p ~/vaultwarden/ssl ~/vaultwarden/vw-data
cd ~/vaultwarden
```

2. Find your host machine's IP address (ip a or hostname -I).

3. Generate certificates inside the ssl directory (replace 192.168.1.100 with your actual IP):

```bash
mkcert -install
mkcert -cert-file ssl/cert.pem -key-file ssl/key.pem localhost 127.0.0.1 ::1 192.168.1.100

```

Note for Windows Users: Since mkcert runs inside WSL, your local Ubuntu environment will trust the certificate. To remove the "Not Secure" browser warning on Windows, copy rootCA.pem to Windows and install it into the Trusted Root Certification Authorities store.

### Step 3: Create docker-compose.yml
In ~/vaultwarden/, create a docker-compose.yml file:

```bash
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      DOMAIN: "[https://172.96.10.153:11001](https://172.96.10.153:11001)"
      ROCKET_TLS: '{certs="/ssl/cert.pem",key="/ssl/key.pem"}'
      SIGNUPS_ALLOWED: "true"
    volumes:
      - ./vw-data:/data
      - ./ssl:/ssl:ro
    ports:
      - 11001:80
```
### Step 4: Run the Container
Start the service using Docker Compose:

```bash
docker compose up -d
```
Access Vaultwarden at: https://localhost:11001 or https://<YOUR-IP>:11001

### 🛠️ Troubleshooting & Solutions
## ❌ Error 1: No such file or directory (os error 2) on TLS files
```bash
error reading TLS file `ssl/cert.pem`: No such file or directory (os error 2)
```
Cause: The certificate files are missing or located outside the ./ssl directory configured in Docker volume mounts.
Fix: Move the generated .pem files inside the ssl/ folder and restart:

```bash
mkdir -p ~/vaultwarden/ssl
mv ~/vaultwarden/cert.pem ~/vaultwarden/ssl/
mv ~/vaultwarden/key.pem ~/vaultwarden/ssl/
docker compose restart
```

## ❌ Error 2: No persistent volume! Warning

```bash
[ERROR] No persistent volume! This will result in permanent data loss...
```

Cause: The /data path inside the container isn't mapped properly to a local folder or the local directory doesn't exist.
Fix: Ensure the ./vw-data directory exists locally and is mapped in docker-compose.yml:

```bash
mkdir -p ~/vaultwarden/vw-data
docker compose down
docker compose up -d
```

## ❌ Error 3: Cannot access via IP address / "Not Secure" warning
Causes & Fixes:
1. Port Mapping: Vaultwarden exposes port 80 internally by default (not 443), so 11001:80 in your docker-compose.yml is correct when Rocket handles TLS internally.

2. Missing SAN (Subject Alternative Name): Re-generate your certificate including all hostnames/IPs you plan to use:

```bash
cd ~/vaultwarden/ssl
mkcert -cert-file cert.pem -key-file key.pem localhost 127.0.0.1 ::1 <YOUR-IP>
```

3. Untrusted Root CA in Windows: mkcert -install inside WSL only installs the CA into Ubuntu's certificate trust store. For Windows browsers (Chrome, Edge) to show the secure lock:

Run mkcert -CAROOT in WSL to locate rootCA.pem.
Copy rootCA.pem to Windows.
Press Win + R, type certmgr.msc, and import rootCA.pem into Trusted Root Certification Authorities.

📚 Credits & Documentation
Vaultwarden Docker Hub - (https://hub.docker.com/r/vaultwarden/server)
Vaultwarden Wiki: Enabling HTTPS (https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS)