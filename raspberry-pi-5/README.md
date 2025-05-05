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
- A Mac/Windows/Linux machine with **SSH enabled** and [Raspberry Pi Imager](https://www.raspberrypi.com/software/) installed  
  (with SD/SSD interface available)  
- *(Optional)* A **Cloudflare account + domain** (for HTTPS and remote access, or use Tailscale Serve if you don't have a domain)

## Setup

### 1. Image the operating system

Open **Raspberry Pi Imager** on your Mac, Windows, or Linux machine:

- **CHOOSE DEVICE**: `Raspberry Pi 5`
- **CHOOSE OS**: `Raspberry Pi OS (other) > Raspberry Pi OS Lite (64-bit)`
- **CHOOSE STORAGE**: Insert your SD card or SSD and select it

Click **Next** > **Edit Settings** and configure:

#### General
- Set **Hostname** to `rpi5` (or your preferred name)
- Set **Username** *(recommend using your PC’s username for consistency)* and a strong password
- *(Optional)* Configure **Wireless LAN** — but a **wired connection** is recommended for stability
- Set **Locale settings** (timezone, keyboard layout)

#### Services
- **Enable SSH**  
  *(Select “Use password authentication” — RSA keys are potentially insecure, will use better encryption)*

Click **Save**, then confirm **Yes** twice. Wait for the writing and verification process to complete.  
Once finished, connect your Pi to power and network, and it will be ready for SSH access.

##
```bash
GUIDE IN PROGRESS
