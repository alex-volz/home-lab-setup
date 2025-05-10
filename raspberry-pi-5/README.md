# Raspberry Pi 5 All‑In‑One Server

> A guide starting from scratch to build a multi-service Raspberry Pi 5 server.

## Description

This guide details the process of hosting multiple services in Docker containers, starting from OS imaging.  
Here’s an overview of the services included in this stack:

- **Pi-hole** — DNS sinkhole and ad blocker  
- **Samba** — File sharing server (Windows-compatible NAS)  
- **Tailscale** — Secure remote network access  
- **Traefik** — Reverse proxy with Cloudflare DNS and HTTPS encryption  
- **Vaultwarden** — Self-hosted password vault (Bitwarden-compatible)  
- **Watchtower** — Automated container updates  

## Prerequisites

- **Raspberry Pi 5** (recommended 8GB RAM model)
- **32GB or larger** SD card *(or preferably SSD)*  
- A Mac/Windows/Linux machine with [Raspberry Pi Imager](https://www.raspberrypi.com/software/) installed  
  (with SD/SSD interface available)  
- *(Optional)* Your own domain **domain** (for HTTPS and remote access)

## Setup

### 1. Image the operating system

Open **Raspberry Pi Imager** on your Mac, Windows, or Linux machine:

- **CHOOSE DEVICE**: `Raspberry Pi 5`
- **CHOOSE OS**: `Raspberry Pi OS (other) > Raspberry Pi OS Lite (64-bit)`
- **CHOOSE STORAGE**: Insert your SD card or SSD and select it

Click **Next** > **Edit Settings** and configure:

#### General
- Set **Hostname** to `rpi5` (or your preferred name) — remember this for later
- Set **Username** *(recommend using your PC’s username for consistency)* and set a strong password
- *(Optional)* Configure **Wireless LAN** — but a **wired connection** is recommended for stability
- Set **Locale settings** (timezone, keyboard layout, etc.)

#### Services
- **Enable SSH**  
  *(Recommended: Select “Allow public-key authentication only”)*

  Instead of logging in with just a password, you can secure SSH access using a key pair generated on your computer.

  **On Windows**:  
  Ensure the built-in SSH client is enabled:  
  - Open **Settings > System > Optional Features**
  - Click **View features**
  - Find and install **OpenSSH Client**

  **Generate an SSH key pair** (Windows, macOS, or Linux):
  1. Open your terminal and run:
     
     ```bash
     ssh-keygen -t ed25519
     ```
  2. When prompted for **"Enter file in which to save the key"**, press **Enter** to accept the default path.
  3. *(Recommended)* Set a passphrase when prompted with **"Enter passphrase"** for extra security.

  Once generated, navigate to your `.ssh` directory:  
  - On Windows: `C:\Users\YourName\.ssh`  
  - On macOS/Linux: `~/.ssh`  

  Open the file ending with `.pub` — this is your **public key**.

  Copy the entire contents of the `.pub` file. Then, in **Raspberry Pi Imager** under:  
  > **Set authorized_keys for 'Username'**

  Paste your public key here.  
  This links your Pi to your private key, enabling secure login **only from computers where your private key (the other file) is stored**.

Click **Save**, then confirm **Yes** twice. Wait for the writing and verification process to complete.  
Once finished, connect your Pi to power and network — it will be ready for SSH access.

### 2. Log in with SSH

Once the Pi finishes booting, open your terminal/shell.

- If you set your username to be the same, log in with:
  
  ```bash
  ssh rpi5
  ```

- Or if it differs, log in with:

  ```bash
  ssh username@rpi5
  ```

If needed enter your **SSH key password**.

If you successfully log in, your terminal will show a prompt like:

  ```ruby
  username@rpi5:~$
  ```

You can easily pull new updates and upgrade your packages in one command with the password you set in the **Raspberry Pi Imager**:

  ```bash
  sudo apt update -y && sudo apt upgrade -y
  ```

**(Optional)**: If your running an NVMe SSD that connects through the PCIe 2.0 x1 interface with a [dedicated M.2 HAT](https://www.microcenter.com/product/671943/5), you can possibly increase the speeds of your drive by enabling PCIe 3.0:

  - Enter the Raspberry Pi config with:

      ```bash
      sudo raspi-config
      ```

  - Using the arrow keys and ```Enter```, navigate to **Advanced Options > PCIe Speed** and choose ```<Yes>```, ```<Ok>```, and ```<Finish>```.

Its a good time to mention that you should reserve a DHCP IP address for your Pi within your [Routers settings](http://192.168.86.1/) so that you can always reach it at the same address.

Once you've assigned your Pi it's own address on your network, see if its using it by typing ```ip addr``` and checking the list.

If it doesn't show the IP you set, try shutting down the Pi with ```sudo systemctl poweroff``` for a couple minutes and then turning it back on, so your router forgets it and reassigns it to the proper address.

### 3. Install DockerCE and Portainer

We'll install DockerCE to host our services locally, using the Debian repository [as instructed by Docker](https://docs.docker.com/engine/install/debian/).

- It consists of pasting a bunch of code into your terminal, and confirming some commands by entering **y**, then logging back in (with SSH):

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
Portainer is a Docker container that hosts a beginner-friendly web UI to manage everything Docker.

- It can be created with one simple command:

  ```bash
  docker run -d --name portainer -p 1100:9000 --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer:/data portainer/portainer-ce:lts
  ```

Now, the Portainer service should be accessible on your local network at [http://rpi5:1100](http://rpi5:1100).

Create a username and password (and optionally disable anayltics), then log in. 

### 3. Create the main container stack

Finally, we have an easily workable container environment we can deploy services to.

```TO BE CONTINUED ```
