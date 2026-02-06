# NVIDIA Jetson Orin Nano (Super Dev Kit)  
# Clean Setup Guide (JetPack 6.2.2)

A clean, repeatable setup for the **NVIDIA Jetson Orin Nano Super Developer Kit**, optimized for:

- Maximum performance mode
- Remote access (VNC)
- Development readiness (SSH, useful tools)
- A “restore point” snapshot

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
- HDMI Monitor  
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

Tip: if you prefer stable hostnames, reserve the Jetson IP in your router’s DHCP settings.

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

XFCE is lighter, faster, and works better with VNC.

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

## Step 8 — Install and Configure VNC (x11vnc)

### Install x11vnc

```bash
sudo apt update
sudo apt install -y x11vnc
```

### Set a VNC password

```bash
sudo x11vnc -storepasswd
```

### Create a systemd service

Replace `<YOUR_USERNAME>` with your Linux username.

```bash
sudo tee /etc/systemd/system/x11vnc.service > /dev/null <<'EOF'
[Unit]
Description=x11vnc remote desktop
After=display-manager.service

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -display :0 -auth /var/run/lightdm/root/:0 \
  -rfbauth /home/<YOUR_USERNAME>/.vnc/passwd \
  -forever -shared -noxdamage -rfbport 5900 -o /var/log/x11vnc.log
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
```

### Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable x11vnc
sudo systemctl start x11vnc
sudo systemctl status x11vnc --no-pager
```

### Open firewall port

```bash
sudo ufw allow 5900/tcp
sudo ufw status
```

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
- Reliable VNC access (x11vnc)
- Chromium working
- Snapshot backup ready

---

## Troubleshooting

See `TROUBLESHOOT.md`.

