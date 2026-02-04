# [FIXED] qBittorrent-nox HTTPS WebUI Not Working

## Problem
qBittorrent-nox WebUI HTTPS fails with `PR_END_OF_FILE_ERROR` in browser. Logs show: `Web UI: HTTPS setup failed, fallback to HTTP`

## Root Cause
qBittorrent 4.4.1 (default Ubuntu 22.04 package) has broken HTTPS auto-certificate generation.

## Solution
Upgrade to qBittorrent 4.5.5+ and provide manual SSL certificate paths.

## Tested Environment
- **OS:** Ubuntu Server 22.04 LTS
- **Hardware:** VPS with 1 vCPU + 2GB RAM
- **qBittorrent Version:** 4.5.5 (from official PPA)

---

## Step-by-Step Fix

### STEP 1: Stop qBittorrent Service

If running as systemd service:
```bash
sudo systemctl stop qbittorrent-nox
```

If running manually:
```bash
sudo pkill qbittorrent-nox
```

---

### STEP 2: Add Official PPA and Upgrade

Add qBittorrent stable PPA:
```bash
sudo add-apt-repository ppa:qbittorrent-team/qbittorrent-stable -y
```

Update package list:
```bash
sudo apt update
```

Upgrade qBittorrent:
```bash
sudo apt upgrade qbittorrent-nox -y
```

Verify version (should be 4.5.5 or newer):
```bash
qbittorrent-nox --version
```

---

### STEP 3: Generate SSL Certificate and Key

Create SSL directory:
```bash
mkdir -p /home/username/.config/qBittorrent/ssl
```

Navigate to directory:
```bash
cd /home/username/.config/qBittorrent/ssl
```

Generate self-signed certificate (replace `YOUR_VPS_IP` with your actual IP address):
```bash
openssl req -x509 -nodes -days 36500 -newkey rsa:2048 -keyout server.key -out server.crt -subj "/C=US/ST=State/L=City/O=Org/CN=YOUR_VPS_IP"
```

This creates:
- `server.key` - Private key
- `server.crt` - Self-signed certificate (valid 100 years)

---

### STEP 4: Edit qBittorrent Configuration

Open configuration file:
```bash
nano /home/username/.config/qBittorrent/qBittorrent.conf
```

Find the `[Preferences]` section and add/modify these lines:
```ini
WebUI\HTTPS\Enabled=true
WebUI\HTTPS\CertificatePath=/home/username/.config/qBittorrent/ssl/server.crt
WebUI\HTTPS\KeyPath=/home/username/.config/qBittorrent/ssl/server.key
WebUI\Port=58080
```

**Note:** Replace `username` with your actual username.

Save the file:
- Press `Ctrl+O`, then `Enter` to save
- Press `Ctrl+X` to exit

---

### STEP 5: Create Systemd Service (Autostart on Boot)

Create service file:
```bash
sudo nano /etc/systemd/system/qbittorrent-nox.service
```

Paste this content (replace `username` with your actual username):
```ini
[Unit]
Description=qBittorrent-nox
After=network.target

[Service]
Type=simple
User=username
ExecStart=/usr/bin/qbittorrent-nox
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

Reload systemd daemon:
```bash
sudo systemctl daemon-reload
```

Enable service to start on boot:
```bash
sudo systemctl enable qbittorrent-nox
```

Start service:
```bash
sudo systemctl start qbittorrent-nox
```

Check service status:
```bash
systemctl status qbittorrent-nox
```

---

### STEP 6: Verify HTTPS is Working

Check qBittorrent logs (should NOT show "HTTPS setup failed"):
```bash
tail -10 /home/username/.local/share/qBittorrent/logs/qbittorrent.log
```

Expected log output should show:
```
(N) YYYY-MM-DD HH:MM:SS - Web UI: Now listening on IP: *, port: 58080
```

Access WebUI:
```
https://YOUR_VPS_IP:58080
```

**Browser Security Warning:** Your browser will show a certificate warning because the certificate is self-signed. This is expected. Click "Advanced" and proceed to access the WebUI.

---

## Firewall Configuration

qBittorrent requires two types of network access:

### 1. WebUI Access (HTTPS)
Allow TCP access to WebUI port:
```bash
sudo ufw allow 58080/tcp
```

### 2. Torrent Traffic (TCP + UDP/uTP)

**Why both TCP and UDP?**
- TCP: Standard BitTorrent protocol
- UDP/uTP (Micro Transport Protocol): Alternative transport with better congestion handling and lower latency

**Port Selection:**
Avoid common BitTorrent ports (6881-6889, 51413) as many ISPs throttle these. Use a random high port in the ephemeral range (49152-65535).

**Example using port 51234:**
```bash
sudo ufw allow 51234/tcp
sudo ufw allow 51234/udp
```

**Configure in qBittorrent:**
1. Access WebUI at `https://YOUR_VPS_IP:58080`
2. Go to **Settings** → **Connection**
3. Set **Port used for incoming connections** to your chosen port (e.g., 51234)
4. Save settings

### UPnP Consideration

**For VPS users:** UPnP not needed - your VPS has a public IP and direct internet access.

**For home users behind NAT/router:**
- UPnP automatically forwards ports through your router
- Enable in qBittorrent: **Settings** → **Connection** → Check "Use UPnP/NAT-PMP port forwarding from my router"
- Requires router to have UPnP enabled
- Still need to configure firewall rules on your local machine

**Security note:** UPnP has security implications. For home use, manually forwarding ports on your router is more secure than enabling UPnP.

### Verify Firewall Rules

Check active UFW rules:
```bash
sudo ufw status numbered
```

Expected output should include:
```
[X] 58080/tcp                  ALLOW IN    Anywhere
[Y] 51234/tcp                  ALLOW IN    Anywhere
[Z] 51234/udp                  ALLOW IN    Anywhere
```

---

## Troubleshooting

### Still getting "HTTPS setup failed"
- Verify certificate files exist:
  ```bash
  ls -l /home/username/.config/qBittorrent/ssl/
  ```
- Check file permissions (should be readable by your user)
- Verify paths in config file are absolute paths (starting with `/home/...`)

### Service won't start
Check logs:
```bash
journalctl -u qbittorrent-nox -n 50
```

### Can't access WebUI
- Verify qBittorrent is listening on the port:
  ```bash
  sudo ss -tlnp | grep 58080
  ```
- Check firewall rules
- Verify VPS security group/firewall allows inbound traffic on port 58080

---

## Why This Works

qBittorrent 4.4.1's automatic certificate generation fails silently. Version 4.5.5+ has improved HTTPS support but still requires manually specified certificate paths. By providing explicit paths to a valid certificate and key, qBittorrent can properly initialize SSL/TLS.

---

## Port Selection Note

Port 58080 is used in this guide because it:
- Falls in the ephemeral/unregistered port range (49152-65535)
- Unlikely to be blocked by ISPs
- No conflicts with registered services

You can use any port in the 49152-65535 range based on preference.
