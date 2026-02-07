# NVIDIA Jetson Orin Nano (Super Dev Kit) — Clean Setup Guide (JetPack 6.2.2)

A clean, repeatable setup for the **NVIDIA Jetson Orin Nano Super Developer Kit**, optimized for:

- Maximum performance mode
- Remote access (VNC) using **TigerVNC** (virtual desktop; works great headless)
- Development readiness (SSH, useful tools)
- A “restore point” snapshot

> This guide intentionally uses **TigerVNC** instead of x11vnc. TigerVNC runs a dedicated virtual desktop at a fixed resolution, so you don’t need a monitor (or a dummy plug) to get a large remote screen.

---

## References

- NVIDIA JetPack Documentation: https://docs.nvidia.com/jetson/jetpack/introduction/index.html  
- balenaEtcher (SD card flashing tool): https://etcher.balena.io/  
- JetPack 6.2.2 download: https://developer.nvidia.com/embedded/jetpack-sdk-622  

---

## Hardware Used (Required)

- NVIDIA Jetson Orin Nano Super Developer Kit: https://amzn.to/4kkrEpl  
- microSD Card (**256GB recommended**): https://amzn.to/3MeFRr2  
- DisplayPort to HDMI Adapter: https://amzn.to/4rxEpPs  
- HDMI Monitor (for initial setup only)  
- USB Keyboard  
- USB Mouse  

---

## Step 1 — Flash JetPack Image to SD Card

1. Download **JetPack 6.2.2**.
2. Download and install **balenaEtcher**.
3. Insert the microSD card into your computer.
4. Open balenaEtcher:
   - Select the JetPack image
   - Select the SD card as target
   - Click **Flash**
5. Wait for flashing + validation to complete.
6. Remove the SD card.

---

## Step 2 — First Boot and Initial Setup

1. Insert the SD card into the Jetson Orin Nano.
2. Connect:
   - Monitor
   - Keyboard
   - Mouse
3. Connect the power supply and power on the Jetson.
4. Follow the setup wizard:
   - Create your user
   - Connect to Wi‑Fi
   - **Do NOT install Chromium when prompted** (we’ll fix it later)

---

## Step 3 — Connect via SSH

1. On the Jetson desktop:
   - **Settings → Wi‑Fi →** note the IP address
2. From your computer (same network):

```bash
ssh <username>@<ip-address>
```

---

## Step 4 — Update the System + Tools

```bash
sudo apt update
sudo apt -y upgrade
sudo apt install -y nano ufw zip unzip
```

---

## Step 5 — Enable Maximum Performance Mode

Check available power modes:

```bash
sudo nvpmodel -q --verbose
```

Set highest performance mode (commonly mode `2`):

```bash
sudo nvpmodel -m 2
```

Verify:

```bash
sudo nvpmodel -q
```

---

## Step 6 — Fix Chromium (Snap Downgrade)

Chromium requires an older `snapd` version.

```bash
sudo snap download snapd --revision=24724
sudo snap ack snapd_24724.assert
sudo snap install snapd_24724.snap
sudo snap refresh --hold snapd
sudo reboot
```

After reboot:

```bash
sudo snap install chromium
```

---

## Step 7 — Switch Desktop from GNOME to XFCE

XFCE is lighter, faster, and works well for remote desktop.

### Install XFCE + LightDM

```bash
sudo apt install -y xfce4 xfce4-goodies lightdm
sudo dpkg-reconfigure lightdm
sudo reboot
```

### Select XFCE Session

At the login screen:
- Click the session icon (gear)
- Choose **Xfce Session**
- Log in

### Remove GNOME (only after confirming XFCE works)

```bash
sudo apt purge -y ubuntu-desktop gnome-shell gdm3
sudo apt autoremove -y
sudo apt autoclean -y
sudo dpkg-reconfigure lightdm
```

---

## Step 8 — Install and Configure VNC (TigerVNC)

TigerVNC provides a dedicated virtual desktop session at a fixed resolution (recommended for headless use).

### 8.1 Install TigerVNC

```bash
sudo apt update
sudo apt install -y tigervnc-standalone-server tigervnc-common
```

### 8.2 Set your VNC password (run as your normal user)

```bash
vncpasswd
```

### 8.3 Configure VNC to start XFCE

Create `~/.vnc/xstartup` (known-good, won’t exit early):

```bash
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup <<'EOF'
#!/bin/sh
# Ignore missing Xresources (common on minimal installs)
xrdb "$HOME/.Xresources" 2>/dev/null || true

# Start XFCE session for the VNC desktop
startxfce4
EOF
chmod +x ~/.vnc/xstartup
```

### 8.4 Start TigerVNC (choose your resolution)

Example (recommended for large monitors):

```bash
vncserver :1 -geometry 2560x1440 -localhost no
```

Connect your VNC client to:
- `JETSON_IP:5901` (display `:1` → port `5900 + 1`)

To stop:

```bash
vncserver -kill :1
```

### 8.5 Auto-start TigerVNC at boot (systemd user service)

Create the service:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/vncserver@:1.service <<'EOF'
[Unit]
Description=TigerVNC server on display %i
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/vncserver %i -geometry 2560x1440 -localhost no
ExecStop=/usr/bin/vncserver -kill %i
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
EOF
```

Enable it and allow it to run after reboot:

```bash
systemctl --user daemon-reload
systemctl --user enable --now vncserver@:1.service
sudo loginctl enable-linger $USER
```

### 8.6 Firewall

```bash
sudo ufw allow 5901/tcp
sudo ufw status
```

> Security note: If possible, prefer an SSH tunnel instead of opening VNC ports. (See TROUBLESHOOT.md.)

---

## Step 9 — Take a System Snapshot (Recommended)

Before continuing development, create a restore point:

```bash
sudo apt install -y timeshift
sudo timeshift --create --comments "Base-Clean-Setup"
```

---

## Result

You now have:

- JetPack 6.2.2 installed
- Maximum performance mode enabled
- XFCE lightweight desktop
- Reliable headless VNC access (TigerVNC)
- Chromium working
- Snapshot backup ready

---

## Troubleshooting

See `TROUBLESHOOT.md`.
