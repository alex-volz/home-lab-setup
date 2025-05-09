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

### 2. Ssh into the rpi

Now that the rpi is booting, its time to log in (give it a moment). Open your preffered terminal (or should i say shell) and ssh in with the command "ssh", followed by the hostname of the device. in my case, it's "ssh rpi5", since i named it that during the install. also, i dont have to specify the username because ssh assumes to use the username of the user executing the command. if using a different username on the system, simply prefix the hostname with "username@", with a full command looking like "ssh username@rpi5"

It will indicate that youve logged in after entering the password by saying a bunch of stuff about linux and giving you a "user@rpi5:~ $" to type in a command. You can update the system and its packages to the latest version in one command by typing "sudo apt update -y && sudo apt upgrade -y", and promptly typing in the password (not the ssh password, but the password you set during imaging).

### 3. Install docker and portainer

Our first thing we'll install is docker, and based on docker's installation instructions (https://docs.docker.com/engine/install/debian/), we will be using the debian image (since the one labelled raspberry pi os/raspbian is for older rpis / rpi operating systems. Basically, paste this into your console and press y and enter when prompted, and log back in after:

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
