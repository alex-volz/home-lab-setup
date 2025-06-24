# Raspberry Pi 5 All‑In‑One Server

> A guide starting from scratch to build a multi-service Raspberry Pi 5 server.

## Description

This guide details the process of hosting multiple services in Docker containers, starting from OS imaging.
Here’s an overview of the services included in this stack:

* **Pi-hole**: DNS sinkhole and ad blocker
* **Samba**: File sharing server (Windows-compatible NAS)
* **Tailscale**: Secure remote network access
* **Traefik**: Reverse proxy with Cloudflare DNS and HTTPS encryption
* **Vaultwarden**: Self-hosted password vault (Bitwarden compatible)
* **Watchtower**: Automated container updates

> **!!!** Keep in mind that you may require different container configuration for your setup, and it is highly recommended you understand each service by reading their documentation.

*This guide is a reference for a working configuration. Some Googling likely required.*

## Prerequisites

* **Raspberry Pi 5** (8 GB or more RAM model recommended)
* **32 GB or larger** SD card *(or preferably an NVMe SSD + HAT)* for the OS
* A Mac/Windows/Linux machine with [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
* Your own **domain** (this guide uses Cloudflare DNS; adapt as needed)

## Setup

### 1. Image the operating system

1. Open **Raspberry Pi Imager** on your main system.
2. **CHOOSE DEVICE**: `Raspberry Pi 5`
3. **CHOOSE OS**: `Raspberry Pi OS (other) > Raspberry Pi OS Lite (64-bit)`
4. **CHOOSE STORAGE**: Select your SD card or SSD

Click **Next** → **Edit Settings**:

#### General

* **Hostname**: `rpi5` (or your preferred)
* **Username**: *(recommend your PC username for consistency)*
* **Password**: strong password
* *(Optional)* **Wireless LAN**: configure if needed; **wired** recommended
* **Locale settings**: timezone, keyboard layout, etc.

#### Services

* **Enable SSH**

  * Choose **Allow public-key authentication only**
  * **Paste your public key** under **Set authorized\_keys for 'Username'**

  **On Windows**:

  1. Go to **Settings > System > Optional Features** → **Add a feature**
  2. Install **OpenSSH Client**

  **Generate SSH key pair** (Windows/macOS/Linux):

  ```bash
  ssh-keygen -t ed25519
  ```

  * Press **Enter** to accept default path
  * *(Optional, but recommended)* set a passphrase

  Find your public key:

  * **Windows**: `C:\Users\YourName\.ssh\id_ed25519.pub`
  * **macOS/Linux**: `~/.ssh/id_ed25519.pub`

  Copy its contents, paste into Imager’s **authorized\_keys** field.

Click **Save**, then confirm **Yes** twice. Wait for imaging to complete. Power on the Pi and connect it to the network.

---

### 2. Log in with SSH

1. In your terminal:

   ```bash
   ssh username@rpi5
   # or if same name:
   ssh rpi5
   ```
2. Enter your SSH key passphrase (if set).

Once in, update and upgrade packages:

```bash
sudo apt update -y && sudo apt upgrade -y
```

> **Optional: NVMe HAT performance unlock**
>
> 1. ```bash
>    sudo raspi-config
>    ```
> 2. Navigate: **Advanced Options > PCIe Speed** → **Yes**, **Ok**, **Finish**

Reserve a static DHCP IP for your Pi in your router settings (e.g., `http://192.168.1.1`).
Verify with:

```bash
ip addr
```

If it doesn’t match, power off (`sudo systemctl poweroff`) for \~2 minutes, then power on again.

---

### 3. Install Docker CE & Portainer

Follow [Docker’s Debian install instructions](https://docs.docker.com/engine/install/debian/):

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo groupadd docker
sudo usermod -aG docker $USER
logout
```

Re-SSH in, then install Portainer:

```bash
docker run -d \
  --name portainer \
  -p 1200:9000 \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer:/data \
  portainer/portainer-ce:lts
```

Access at [http://rpi5:1200](http://rpi5:1200) or `http://<Pi_IP>:1200`. Create admin user and optionally disable analytics.

---

### 4. Create the main container stack

1. In Portainer: **Live connect** → **Stacks** → **Add stack**
2. **Name** your stack (e.g., `server-stack`).
3. **Load variables** from `.env` file.
4. **Paste** the `docker-compose.yml` into the Web editor:

```yaml
services:
  homepage:
    container_name: homepage
    image: ghcr.io/gethomepage/homepage:latest
    volumes:
      - /home/alexg/homepage:/app/config
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      HOMEPAGE_ALLOWED_HOSTS: hp.${DOMAIN}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.hp.rule=Host(`hp.${DOMAIN}`)"
      - "traefik.http.routers.hp.entrypoints=websecure"
      - "traefik.http.routers.hp.tls.certresolver=letsencrypt"
      - "traefik.http.services.hp.loadbalancer.server.port=3000"
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - 53:53/tcp
      - 53:53/udp
    environment:
      TZ: '${TIMEZONE}'
      FTLCONF_webserver_api_password: '${PIHOLE_PASSWORD}'
      FTLCONF_dns_listeningMode: 'all'
    volumes:
      - pihole:/etc/pihole
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ph.rule=Host(`ph.${DOMAIN}`)"
      - "traefik.http.routers.ph.entrypoints=websecure"
      - "traefik.http.routers.ph.tls.certresolver=letsencrypt"
      - "traefik.http.services.ph.loadbalancer.server.port=80"
  samba:
    container_name: samba
    image: ghcr.io/servercontainers/samba:latest
    restart: unless-stopped
    network_mode: host
    cap_add:
      - CAP_NET_ADMIN
    environment:
      SAMBA_CONF_LOG_LEVEL: 3
      AVAHI_DISABLE: 1
      ACCOUNT_${SAMBA_USERNAME_1}: "${SAMBA_PASSWORD_1}"
      UID_${SAMBA_USERNAME_1}: 1000
      ACCOUNT_${SAMBA_USERNAME_2}: "${SAMBA_PASSWORD_2}"
      UID_family: 1001
      SAMBA_VOLUME_CONFIG_share: "[${SAMBA_USERNAME_1}]; path=/shares/${SAMBA_USERNAME_1}; valid users = ${SAMBA_USERNAME_1}; writeable = yes; write list = ${SAMBA_USERNAME_1}; create mask = 0775; directory mask = 0775"
      SAMBA_VOLUME_CONFIG_family: "[${SAMBA_USERNAME_2}]; path=/shares/${SAMBA_USERNAME_2}; valid users = ${SAMBA_USERNAME_1}, ${SAMBA_USERNAME_2}; force user = ${SAMBA_USERNAME_2}; writeable = yes; write list = ${SAMBA_USERNAME_2}; create mask = 0774; directory mask = 0774"
      SAMBA_VOLUME_CONFIG_${SAMBA_USERNAME_2}r: "[${SAMBA_USERNAME_2} - Read Only]; path=/shares/${SAMBA_USERNAME_2}; guest ok = yes"
    volumes:
      - /shares:/shares
      - /shares/${SAMBA_USERNAME_1}:/shares/${SAMBA_USERNAME_1}
      - /shares/${SAMBA_USERNAME_2}:/shares/${SAMBA_USERNAME_2}
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    environment:
      - TS_HOSTNAME=rpi5
      - TS_AUTHKEY=${TAILSCALE_KEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false # Optional: force kernel-based networking; userspace fallback if not supported
      - TS_ROUTES=${SUBNET}
      - TS_EXTRA_ARGS=--advertise-exit-node
    volumes:
      - tailscale:/var/lib/tailscale
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
      - sys_module
    restart: unless-stopped
    network_mode: host
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      - CLOUDFLARE_DNS_API_TOKEN=${CLOUDFLARE_TOKEN}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik:/letsencrypt
    command:
      - "--api=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.websecure.address=:443"
      - "--entryPoints.web.http.redirections.entrypoint.to=websecure"
      - "--entryPoints.web.http.redirections.entrypoint.scheme=https"
      - "--certificatesresolvers.letsencrypt.acme.email=${CLOUDFLARE_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.dnschallenge.provider=cloudflare"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tf.rule=Host(`tf.${DOMAIN}`)"
      - "traefik.http.routers.tf.service=api@internal"
      - "traefik.http.routers.tf.entrypoints=websecure"
      - "traefik.http.routers.tf.tls.certresolver=letsencrypt"
      - "traefik.http.services.tf.loadbalancer.server.port=8080"
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
#      - ADMIN_TOKEN=${VAULTWARDEN_ADMIN}
      - SIGNUPS_ALLOWED=false
      - INVITATIONS_ALLOWED=false
    volumes:
      - vaultwarden:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vw.rule=Host(`vw.${DOMAIN}`)"
      - "traefik.http.routers.vw.entrypoints=websecure"
      - "traefik.http.routers.vw.tls.certresolver=letsencrypt"
      - "traefik.http.services.vw.loadbalancer.server.port=80"
      - "com.centurylinklabs.watchtower.monitor-only=true"
  watchtower:
    container_name: watchtower
    image: containrrr/watchtower:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_NOTIFICATIONS=email
      - WATCHTOWER_NOTIFICATION_EMAIL_FROM=${NOTIFICATION_EMAIL}
      - WATCHTOWER_NOTIFICATION_EMAIL_TO=${NOTIFICATION_EMAIL}
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER=smtp.gmail.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT=587
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER=${NOTIFICATION_EMAIL}
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD=${GMAIL_APP_PASSWORD}
      - WATCHTOWER_CLEANUP=true
volumes:
  pihole:
  tailscale:
  traefik:
  vaultwarden:
```

5. Load the .env file, edit the values, and click **Deploy the stack**.

---

## `.env` File

```env
#Your domain that points to your Raspberry Pi 5's IP
DOMAIN=yourdomain.com

#Your timezone (TZ database standard)
TIMEZONE=America/New_York

#A password for the Pihole dashboard
PIHOLE_PASSWORD=password

#Usernames/passwords for 2 Samba shares
SAMBA_USERNAME_1=username1
SAMBA_PASSWORD_1=password1
SAMBA_USERNAME_2=username2
SAMBA_PASSWORD_2=password2

#Tailscale auth key (tskey-...)
TAILSCALE_KEY=authkey

#Local subnet for routing
SUBNET=192.168.1.0/24

#Cloudflare API token & email
CLOUDFLARE_TOKEN=abcdefghijklmnopqrstuvwxyz
CLOUDFLARE_EMAIL=account@emailprovider.com

#Vaultwarden admin token
VAULTWARDEN_ADMIN=password

#Email for Watchtower notifications
NOTIFICATION_EMAIL=account@gmail.com

#Gmail app password for SMTP
GMAIL_APP_PASSWORD=password
```

*Refer to comments for explanations of each variable.*

---

## Post-Deployment Tasks

1. **Portainer via Traefik (HTTPS):**

Portainer can be served securely over HTTPS now that Traefik is set up, paste these commands:
   ```bash
   docker stop portainer && docker rm portainer
  
   docker run -d --name portainer -p 1200:9000 \
     --restart always --network server_default \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v portainer:/data \
     -l "traefik.enable=true" \
     -l "traefik.http.routers.pt.rule=Host(`pt.${DOMAIN}`)" \
     -l "traefik.http.routers.pt.entrypoints=websecure" \
     -l "traefik.http.routers.pt.tls.certresolver=letsencrypt" \
     -l "traefik.http.services.pt.loadbalancer.server.port=9000" \
     portainer/portainer-ce:lts
   ```

2. **Tailscale Admin:** disable key expiry for your devices, and approve subnet routes so you can connect to any device on your home network remotely, not just your server..

3. **Pi-hole UI:** visit `ph.${DOMAIN}`, and optionally block any extra domains.

4. **Vaultwarden:** uncomment `ADMIN_TOKEN`, set `SIGNUPS_ALLOWED=true`, visit `vw.${DOMAIN}`, create account, then revert settings and remove token to disable the admin panel again:

   ```bash
   sed -i '/"admin_token":/d' /data/config.json
   ```

5. **Samba Permissions:**

In order to be able to write to the Samba shares, the permissions should be changed.

   ```bash
   sudo useradd username1 && sudo useradd username2
   sudo chown -R username1 /shares/username1
   sudo chown -R username2 /shares/username2
   ```

> You now have a fully functional, multi-service Raspberry Pi 5 server.
