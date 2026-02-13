# Jetson Orin Nano Super (JetPack 6.x) — Ubuntu + XFCE + Persistent VNC + Firefox + Cursor

This guide captures the **working** setup and the fixes discovered during troubleshooting:
- XFCE desktop
- **Persistent TigerVNC** that survives logout/reboot
- **Firefox via Flatpak** (Snap is broken on this kernel)
- Cursor installed and runnable, with login troubleshooting

---

## 1) Baseline: Confirm platform

```bash
uname -a
lsb_release -a || cat /etc/os-release
```

---

## 2) Install XFCE + LightDM (if not already)

```bash
sudo apt update
sudo apt install -y xfce4 xfce4-goodies lightdm
```

Set LightDM as default display manager:

```bash
sudo dpkg-reconfigure lightdm
```

Reboot:

```bash
sudo reboot
```

---

## 3) Install TigerVNC server

```bash
sudo apt update
sudo apt install -y tigervnc-standalone-server tigervnc-common
```

---

## 4) Configure TigerVNC session to launch XFCE

Create the VNC startup script (user: luna):

```bash
mkdir -p /home/luna/.vnc
cat <<'EOF' > /home/luna/.vnc/xstartup
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
EOF
chmod +x /home/luna/.vnc/xstartup
```

Optional: set VNC config (recommended for remote access):

```bash
cat <<'EOF' > /home/luna/.vnc/config
localhost=no
EOF
```

Set the VNC password (required by many VNC viewers):

```bash
vncpasswd
```

---

## 5) Make TigerVNC persistent via systemd (survives logout)

### 5.1 Map display to user

```bash
sudo mkdir -p /etc/tigervnc
printf '%s\n' ':1=luna' | sudo tee /etc/tigervnc/vncserver.users >/dev/null
```

### 5.2 Fix systemd SELinuxContext failure (Jetson typically has no SELinux)

TigerVNC unit includes an SELinuxContext line that can break service start on Jetson.
Create a systemd override to remove it:

```bash
sudo systemctl edit tigervncserver@.service
```

Paste:

```ini
[Service]
SELinuxContext=
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

### 5.3 Start and enable the correct instance

Use **@:1** format (this matters):

```bash
sudo systemctl enable --now tigervncserver@:1.service
```

Verify:

```bash
systemctl status tigervncserver@:1.service --no-pager -l
sudo ss -ltnp | egrep ":5901" || true
```

Expected: `0.0.0.0:5901` or `127.0.0.1:5901` depending on config.

---

## 6) Connecting to VNC

### Option A (recommended): SSH tunnel + local-only VNC
If you keep VNC bound to localhost, tunnel it:

On your laptop/desktop:

```bash
ssh -L 5901:127.0.0.1:5901 luna@<JETSON_IP>
```

Then in your VNC viewer connect to:

- `127.0.0.1:5901`

### Option B: Direct remote VNC
If you set `localhost=no` in `~/.vnc/config`, connect directly to:

- `<JETSON_IP>:5901`

---

## 7) Browser: Snap is NOT supported (use Flatpak Firefox)

### Why
On this Jetson kernel, Snap frequently fails with SquashFS mount errors like:
- `wrong fs type, bad superblock on /dev/loop0`
- snap mounts failing for chromium/firefox

Do **not** use Snap for browsers on this setup.

### Install Flatpak + Firefox (user-scoped)

```bash
sudo apt update
sudo apt install -y flatpak
flatpak --user remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub org.mozilla.firefox
```

Run:

```bash
flatpak run --user org.mozilla.firefox
```

---

## 8) Desktop shortcut for Flatpak Firefox (so it appears in XFCE)

Sometimes icons do not appear until you add XDG paths and restart session.

### 8.1 Export Flatpak desktop entries to XFCE menus
Create `~/.profile` entries (safe on XFCE):

```bash
grep -q 'flatpak/exports/share' ~/.profile || cat <<'EOF' >> ~/.profile

# Flatpak user exports for desktop entries/icons
export XDG_DATA_DIRS="$HOME/.local/share/flatpak/exports/share:/var/lib/flatpak/exports/share:${XDG_DATA_DIRS:-/usr/local/share:/usr/share}"
EOF
```

Log out/in (or reboot).

### 8.2 Optional: create desktop launcher explicitly
```bash
mkdir -p ~/Desktop
cat <<'EOF' > ~/Desktop/Firefox.desktop
[Desktop Entry]
Type=Application
Name=Firefox (Flatpak)
Exec=flatpak run --user org.mozilla.firefox
Icon=org.mozilla.firefox
Terminal=false
Categories=Network;WebBrowser;
EOF
chmod +x ~/Desktop/Firefox.desktop
```

---

## 9) Cursor

### 9.1 Verify install
Depending on how you installed Cursor, verify binary exists:

```bash
command -v cursor || true
which cursor || true
```

If Cursor is a snap: **avoid snap** (same reasons as browsers). Prefer a .deb/AppImage if available.

### 9.2 Cursor login button does nothing (common)
Fix path:
1) Ensure Firefox opens (Flatpak)
2) Open Cursor from a clean VNC session (after reboot/login)
3) Try login again
4) If still stuck, launch Cursor from terminal and capture logs:

```bash
cursor --verbose 2>&1 | tee ~/cursor-login.log
```

---

## 10) Notes on warnings you may see

### 10.1 “System policy prevents Wi-Fi scans”
Expected in headless/VNC sessions. Harmless. Wi-Fi generally still works.

### 10.2 polkit agent “already exists”
Harmless. Happens when multiple sessions try to register an auth agent inside VNC.

---

## 11) Quick health checks (copy/paste)

```bash
# VNC service health
systemctl is-active tigervncserver@:1.service
systemctl status tigervncserver@:1.service --no-pager -l

# VNC port listening
sudo ss -ltnp | egrep ":5901" || true

# Flatpak Firefox installed
flatpak --user list | grep -i firefox || true
```

---

## 12) Reboot-safe workflow

- Reboot Jetson
- Connect VNC to :1 (port 5901)
- Launch Firefox via Flatpak
- Launch Cursor
- Do NOT rely on Snap for GUI apps
