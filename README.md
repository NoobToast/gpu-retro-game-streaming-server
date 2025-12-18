# GPU-Accelerated Docker Game Streaming Service

Stream retro games from a headless Linux server to thin clients anywhere on your network.

![Docker](https://img.shields.io/badge/docker-required-blue)
![NVIDIA](https://img.shields.io/badge/NVIDIA-GPU-green)
![Ubuntu](https://img.shields.io/badge/ubuntu-24.04-orange)

## Overview

A complete infrastructure setup for GPU-accelerated game streaming using:
- **Wolf** (Games on Whales) for session management
- **NVIDIA NVENC** for hardware encoding
- **RetroArch** containers for emulation
- **Moonlight** thin clients for low-latency decoding
- **Dell Wyse 5070** thin clients configured as dedicated gaming terminals

Built on Ubuntu Server with Docker, pushing frames directly from an NVIDIA GPU to Moonlight clients anywhere on your network.

## Table of Contents
- [Why I Built This](#why-i-built-this)
- [Hardware Requirements](#environment-assumptions)
- [Target State](#-final-target-state-success-criteria)
- [Phase 1: NVIDIA + Docker GPU Foundation](#-phase-1--nvidia--docker-gpu-foundation)
- [Phase 2: Wolf Setup](#-phase-2--wolf-headless-nvidia-stable)
- [Phase 3: Moonlight Thin Client](#️-phase-2--dell-wyse-5070-moonlight-thin-client)
- [Phase 4: RetroArch Configuration](#-phase-3--retroarch--n64)
- [Troubleshooting](#troubleshooting)
- [Expanding Beyond RetroArch](#expanding-beyond-retroarch)
- [Contributing](#contributing)

## Why I Built This

It started on a quiet Friday night with me and my cats. I was sitting there, looking at my old retro game collection on my bookshelf — cartridges and discs from the early '90s through the early 2000s. You know the good ol' days… my youth. Nintendo 64. PlayStation. Atari. Super Nintendo. Game Boy Color. The classics. Super Mario 64. Majora's Mask. Super Mario Bros. Sonic the Hedgehog.

I assumed all of them still worked, but I didn't have a console to play them on. I could have solved that the easy way: buy a used N64, hunt down cables, deal with aging hardware and finicky controllers — really take me back and blow out the cartridge. Or I could have done the boring solution: install an emulator on my desktop and call it a day.

But I wanted something more. I wanted to play the games, but they're light enough that I wanted them available anywhere. I wanted something that felt modern, flexible, and a little over-engineered. Something I could learn from. Something reusable. Something I could build once and expand forever.

That's when I decided my first serious Docker project would be a GPU-accelerated game streaming server: a headless Linux server running containers, pushing frames directly from an NVIDIA GPU to Moonlight clients anywhere on my network.

## 📘 Purpose

Establish a fully functional NVIDIA GPU + Docker environment capable of running:
- 🐺 Wolf (Games on Whales)
- ☀️ Moonlight on a Dell Wyse 5070
- 🎮 RetroArch and other GPU-accelerated containers

This document validates the GPU stack first, then builds a known-good Wolf → Moonlight → RetroArch path. If the foundation is correct, everything above it works reliably.

## 🧱 Final Target State (Success Criteria)

At the end of this guide, all of the following must be true:
- ✅ NVIDIA driver works on the host
- ✅ Docker installed from the official Docker repository
- ✅ NVIDIA Container Toolkit installed and configured
- ✅ Docker can access the GPU via `--gpus all`
- ✅ `nvidia-smi` works inside a container
- ✅ Wolf runs headless (no X11 / Wayland)
- ✅ Moonlight reconnects reliably
- ✅ RetroArch runs with GPU acceleration and visible ROMs

⚠️ **If any of these fail, Wolf / Moonlight will not work correctly.**

## 🧭 Environment Assumptions

| Component | Value |
|-----------|-------|
| OS | Ubuntu 24.04 (Noble) |
| GPU | NVIDIA GTX 1050 Ti |
| CPU | Ryzen 5 3600 |
| RAM | 16 GB |
| User | greg |
| Docker | Official Docker repo |
| Secure Boot | Disabled or not blocking NVIDIA modules |

---

## 🧩 PHASE 1 — NVIDIA + Docker GPU Foundation

### 1️⃣ Verify NVIDIA Driver on Host

Before Docker is involved, the GPU must work natively.

```bash
nvidia-smi
```

✅ **Expected:**
- GTX 1050 Ti visible
- Driver version shown

🚫 **If this fails — stop here.**

### 2️⃣ Install Docker Engine (Official Method)

Ubuntu's default Docker packages are too old and break GPU support.

#### 2.1 Remove conflicting packages

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

#### 2.2 Install prerequisites

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

#### 2.3 Add Docker GPG key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

#### 2.4 Add Docker repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### 2.5 Install Docker Engine + Compose

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

#### 2.6 Enable Docker for non-root user

```bash
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

### 3️⃣ Confirm GPU Is Not Available Yet (Expected Failure)

```bash
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

❌ **Expected error** confirms NVIDIA runtime is not installed yet.

### 4️⃣ Install NVIDIA Container Toolkit (Ubuntu 24.04)

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 5️⃣ Final Proof: GPU Inside Docker

```bash
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

🎉 **GPU-accelerated Docker is now fully functional.**

---

## 🐺 PHASE 2 — Wolf (Headless, NVIDIA, Stable)

### 🎯 Goal
- Wolf running headless
- No desktop environment
- Stable Moonlight reconnects

### 1️⃣ Create Wolf Directory Structure

```bash
mkdir -p ~/wolf
mkdir -p /etc/wolf/cfg
cd ~/wolf
```

### 2️⃣ Minimal Known-Good docker-compose.yml

```yaml
services:
  wolf:
    image: ghcr.io/games-on-whales/wolf:stable
    container_name: wolf
    network_mode: host
    restart: unless-stopped
    volumes:
      - /etc/wolf:/etc/wolf
      - /var/run/docker.sock:/var/run/docker.sock
      - /dev:/dev
      - /run/udev:/run/udev
    devices:
      - /dev/dri
      - /dev/uinput
      - /dev/uhid
      - /dev/nvidia0
      - /dev/nvidiactl
      - /dev/nvidia-modeset
      - /dev/nvidia-uvm
      - /dev/nvidia-uvm-tools
      - /dev/nvidia-caps/nvidia-cap1
      - /dev/nvidia-caps/nvidia-cap2
    device_cgroup_rules:
      - 'c 13:* rmw'
```

### 3️⃣ Start Wolf

```bash
docker compose up -d
docker ps
```

### 4️⃣ Verify Wolf Logs (Critical)

```bash
docker logs -f wolf
```

✅ **Expected indicators:**
- NVENC selected
- Zero-copy pipeline enabled
- RTSP / control ports listening

### 5️⃣ Verify Ports

```bash
sudo ss -tulnp | grep -E '47984|47989|47999|48010'
```

---

## 🖥️ PHASE 2 — Dell Wyse 5070 Moonlight Thin Client

### 🎯 Goal

Convert a Dell Wyse 5070 into a dedicated Moonlight terminal that:
- Boots automatically
- Connects directly to Wolf
- Launches Wolf UI immediately
- Exposes no desktop
- Shuts down cleanly when the session ends

### Client Hardware

| Component | Value |
|-----------|-------|
| Device | Dell Wyse 5070 |
| CPU | Intel Gemini Lake |
| GPU | Intel iGPU (VAAPI decode) |
| Storage | 64–128 GB SSD |
| Network | Wired Ethernet |

### Client OS
- Ubuntu Desktop 22.04 / 24.04
- Desktop required for graphics/audio/input
- Desktop UI will be hidden (kiosk mode)

### 1️⃣ Install Flatpak

```bash
sudo apt update
sudo apt install -y flatpak
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

Log out and back in.

### 2️⃣ Install Moonlight (Flatpak)

```bash
flatpak install -y flathub com.moonlight_stream.Moonlight
```

Verify:
```bash
flatpak list | grep moonlight
```

### 3️⃣ Pair Moonlight with Wolf

```bash
flatpak run com.moonlight_stream.Moonlight pair <SERVER_IP>
```

Watch Wolf logs for the PIN:
```bash
docker logs -f wolf
```

Verify:
```bash
flatpak run com.moonlight_stream.Moonlight list <SERVER_IP>
```

Expected: `Wolf UI`

### 4️⃣ Auto-Launch Wolf UI (Skip Picker)

```bash
nano ~/start-moonlight.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

flatpak run com.moonlight_stream.Moonlight stream <SERVER_IP> "Wolf UI"
sleep 2
systemctl poweroff
```

```bash
chmod +x ~/start-moonlight.sh
```

### 5️⃣ Enable Auto-Login

```bash
sudo nano /etc/gdm3/custom.conf
```

```ini
[daemon]
AutomaticLoginEnable=true
AutomaticLogin=retro-gaming-client
```

### 6️⃣ Auto-Start Moonlight

```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/moonlight.desktop
```

```ini
[Desktop Entry]
Type=Application
Name=Moonlight
Exec=/home/retro-gaming-client/start-moonlight.sh
Terminal=false
X-GNOME-Autostart-enabled=true
```

### 7️⃣ Lock Down Desktop (Kiosk Mode)

```bash
gsettings set org.gnome.desktop.session idle-delay 0
gsettings set org.gnome.desktop.screensaver lock-enabled false
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
gsettings set org.gnome.mutter overlay-key ''
gsettings set org.gnome.desktop.wm.keybindings switch-applications "[]"
gsettings set org.gnome.desktop.wm.keybindings switch-windows "[]"
gsettings set org.gnome.desktop.interface enable-hot-corners false
gsettings set org.gnome.shell.extensions.dash-to-dock autohide true
```

---

## ⚙️ PHASE 3 — Moonlight Client Configuration

Recommended settings:
- **Resolution:** match display (1080p)
- **FPS:** 60
- **Codec:** H.264
- **Decoder:** Hardware (VAAPI)
- **Bitrate:** 15–20 Mbps (LAN)
- **VSync:** Off
- **HDR:** Off
- **Adaptive Bitrate:** On

Moonlight handles decode only. Wolf + NVENC handle render + encode.

---

## 🎮 PHASE 3 — RetroArch + N64

### 1️⃣ ROM Layout

```
/500GBStorage/ROM_Collection/
└── N64_Collection/
    ├── Banjo-Kazooie.z64
    ├── Super Smash Bros.z64
    └── Zelda - OOT.z64
```

### 2️⃣ Add RetroArch App to Wolf

```bash
sudo nano /etc/wolf/cfg/config.toml
```

```toml
[[profiles.apps]]
title = "RetroArch"
start_virtual_compositor = true

[profiles.apps.runner]
type = "docker"
name = "WolfRetroArch"
image = "ghcr.io/games-on-whales/retroarch:edge"
mounts = [
  "/500GBStorage/ROM_Collection:/roms:ro"
]
env = [
  "RUN_SWAY=1",
  "GOW_REQUIRED_DEVICES=/dev/input/* /dev/dri/* /dev/nvidia*"
]
```

### 3️⃣ Restart Wolf

```bash
docker restart wolf
docker logs -f wolf
```

### 4️⃣ Verify ROM Mount

```bash
docker exec -it WolfRetroarch_* sh -lc "ls -la /roms/N64_Collection"
```

### 5️⃣ RetroArch Configuration

- Disable "Filter Unknown Extensions"
- Set Content Directory to `/roms`
- Enable Network + Updater
- Install Mupen64Plus-Next core

---

## 🏁 Final Result

- ✅ Headless GPU server
- ✅ On-demand containerized apps
- ✅ Dell Wyse thin-client terminals
- ✅ Appliance-grade boot behavior
- ✅ Clean session lifecycle

---

## Troubleshooting

### GPU not detected in container

```bash
# Verify nvidia-smi works on host first
nvidia-smi

# Check NVIDIA runtime configuration
sudo nvidia-ctk runtime configure --runtime=docker

# Restart Docker daemon
sudo systemctl restart docker
```

### Moonlight won't connect

```bash
# Verify ports are listening
sudo ss -tulnp | grep -E '47984|47989|47999|48010'

# Check Wolf logs
docker logs -f wolf

# Ensure client and server are on same network/VLAN
```

### RetroArch ROMs not visible

```bash
# Verify mount inside container
docker exec -it WolfRetroarch_* sh -lc "ls -la /roms"

# Check permissions on host ROM directory
ls -la /500GBStorage/ROM_Collection

# In RetroArch: Settings → Directory → disable "Filter Unknown Extensions"
```

### Wolf container won't start

```bash
# Check device permissions
ls -la /dev/nvidia*
ls -la /dev/dri

# Verify user is in docker group
groups $USER

# Check for port conflicts
sudo ss -tulnp | grep -E '47984|47989|47999|48010'
```

---

## Expanding Beyond RetroArch

This architecture supports any GPU-accelerated application. Example configurations:

### Steam

```toml
[[profiles.apps]]
title = "Steam"
start_virtual_compositor = true

[profiles.apps.runner]
type = "docker"
name = "WolfSteam"
image = "ghcr.io/games-on-whales/steam:edge"
```

### Potential Additions

- **Dolphin** (GameCube/Wii)
- **RPCS3** (PS3)
- **PCSX2** (PS2)
- **Yuzu/Ryujinx** (Switch)
- **Steam/Proton** games
- **MAME** (Arcade)

Each runs in its own container with isolated configuration.

---

## Contributing

Issues and pull requests welcome! Particularly interested in:
- Additional emulator configurations
- Alternative thin client setups (Raspberry Pi, old laptops, etc.)
- Performance optimization tips
- Network/security hardening

---

## License

MIT License - feel free to use this for your own setup.

---

## Acknowledgments

Built with assistance from ChatGPT for Docker/NVIDIA configuration troubleshooting.

Architecture, testing methodology, and integration work developed through hands-on experimentation.

Special thanks to:
- [Games on Whales](https://github.com/games-on-whales) for Wolf
- [Moonlight](https://moonlight-stream.org/) for the streaming client
- [RetroArch](https://www.retroarch.com/) for the emulation frontend

---

**Author:** Greg Diny  
**Date:** December 15, 2025  
**Built with:** Coffee, nostalgia, and a couple of cats 🐱🎮
