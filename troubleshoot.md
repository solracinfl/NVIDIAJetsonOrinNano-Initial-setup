# TROUBLESHOOT — Jetson Orin Nano Super (JetPack 6.x) XFCE + VNC + Firefox + Cursor

---

## 1) VNC works after reboot, but dies after logout

**Symptoms**
- VNC connects after reboot
- Logging out causes “server refused connection”
- You must reboot to recover

**Root cause**
- VNC was tied to an interactive user session instead of a system-managed service

**Fix**
Use systemd-managed TigerVNC instance `@:1`.

```bash
sudo systemctl enable --now tigervncserver@:1.service
systemctl status tigervncserver@:1.service --no-pager -l
sudo ss -ltnp | egrep ":5901" || true
```

If the port is only on localhost and you need remote access:
- set `localhost=no` in `~/.vnc/config`
- restart service

```bash
echo "localhost=no" > ~/.vnc/config
sudo systemctl restart tigervncserver@:1.service
sudo ss -ltnp | egrep ":5901" || true
```

---

## 2) TigerVNC service fails with SELINUX_CONTEXT

**Symptoms**
- `Failed at step SELINUX_CONTEXT`
- `Operation not permitted`

**Root cause**
- Unit file contains `SELinuxContext=...` but Jetson typically has no SELinux

**Fix**
Remove SELinuxContext with a systemd override:

```bash
sudo systemctl edit tigervncserver@.service
```

Add:

```ini
[Service]
SELinuxContext=
```

Reload + restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart tigervncserver@:1.service
```

---

## 3) TigerVNC says “No user configured for display 1”

**Symptoms**
- `No user configured for display 1`

**Root cause**
- Missing or wrong `/etc/tigervnc/vncserver.users` mapping

**Fix**
For `tigervncserver@:1.service`, the mapping must be `:1=luna`

```bash
sudo mkdir -p /etc/tigervnc
printf '%s\n' ':1=luna' | sudo tee /etc/tigervnc/vncserver.users >/dev/null
sudo systemctl restart tigervncserver@:1.service
```

---

## 4) Snap apps (chromium/firefox/cursor) fail or won’t mount

**Symptoms**
- `wrong fs type, bad superblock on /dev/loop0`
- snap mount unit fails
- `system does not fully support snapd: cannot mount squashfs`
- apps complain about snap cgroups

**Root cause**
- SquashFS is not available/loaded/supported for snap mounts on this kernel

**Fix**
Stop using snap for GUI apps. Use Flatpak for browser. Use .deb/AppImage for editor if possible.

### Flatpak Firefox (works)

```bash
sudo apt update
sudo apt install -y flatpak
flatpak --user remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub org.mozilla.firefox
flatpak run --user org.mozilla.firefox
```

---

## 5) Firefox runs but no desktop icon appears

**Symptoms**
- Firefox runs from terminal
- No menu entry or desktop icon

**Fix**
Add Flatpak exports to `XDG_DATA_DIRS`, then relogin.

```bash
grep -q 'flatpak/exports/share' ~/.profile || cat <<'EOF' >> ~/.profile

export XDG_DATA_DIRS="$HOME/.local/share/flatpak/exports/share:/var/lib/flatpak/exports/share:${XDG_DATA_DIRS:-/usr/local/share:/usr/share}"
EOF
```

Create a desktop shortcut:

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

## 6) XFCE Terminal won’t open, and you are on SSH

**Symptoms**
- `Gtk-WARNING: cannot open display`
- commands fail because you are not on a GUI session

**Explanation**
From SSH, you cannot launch GUI apps unless you export DISPLAY and have X authority. Prefer running GUI apps inside VNC, not via SSH.

---

## 7) “System policy prevents Wi‑Fi scans” popup

**Status**
Expected in headless/VNC sessions.

**Fix**
None required. If Wi‑Fi is functioning, ignore it.

---

## 8) polkit agent “already exists”

**Status**
Harmless. Caused by multiple sessions registering polkit agent.

**Fix**
None required unless you see repeated password prompts or blocked admin actions.

---

## 9) Quick log commands

```bash
# VNC logs
journalctl -xeu tigervncserver@:1.service --no-pager | tail -n 200

# Display manager logs
sudo tail -n 200 /var/log/lightdm/lightdm.log

# Auth logs
sudo egrep -i "lightdm|pam|authentication failure|Failed" /var/log/auth.log | tail -n 120
```
