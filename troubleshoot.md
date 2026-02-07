# Troubleshooting — Jetson Orin Nano Clean Setup (JetPack 6.2.2)

This file covers common problems when setting up **SSH**, **Chromium**, **XFCE**, and **VNC (TigerVNC)**.

---

## SSH Issues

### "Connection timed out" / "No route to host"

- Confirm the Jetson IP address:

```bash
hostname -I
ip a
```

- Confirm SSH server is installed and running:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

- Confirm port 22 is listening:

```bash
sudo ss -tulpn | grep :22
```

---

## Performance Mode (nvpmodel)

### "nvpmodel: command not found"

JetPack should include it, but if missing:

```bash
sudo apt update
sudo apt install -y nvpmodel
```

If that package doesn’t exist, your JetPack install may be incomplete.

---

## Chromium / Snap Issues

### Chromium still won’t install

- Confirm snapd is held:

```bash
snap list | grep snapd
sudo snap refresh --time
```

- Retry install:

```bash
sudo snap install chromium
```

---

## XFCE / Login Issues

### XFCE option not visible at login

Ensure LightDM is selected:

```bash
sudo dpkg-reconfigure lightdm
sudo reboot
```

---

## TigerVNC Issues

### VNC starts then immediately stops / connection refused

If your VNC client says "refused" but you can SSH, the VNC server likely started and then exited. This is commonly caused by `~/.vnc/xstartup` exiting too early.

If the server stays running but you only see a gray/blank desktop, it’s still usually an `xstartup` issue—apply the `xstartup` fix below and restart the server.

1) Check the log:

```bash
ls -la ~/.vnc
# replace host/port in filename as needed
tail -n 200 ~/.vnc/*.log
```

2) Install required packages (usually already installed, but safe):

```bash
sudo apt install -y dbus-x11 xfce4 xfce4-goodies
```

3) Use this known-good `~/.vnc/xstartup`:

```bash
cat > ~/.vnc/xstartup <<'EOF'
#!/bin/sh
xrdb "$HOME/.Xresources" 2>/dev/null || true
startxfce4
EOF
chmod +x ~/.vnc/xstartup
```

4) Restart the server cleanly:

```bash
vncserver -kill :1 || true
vncserver :1 -geometry 2560x1440 -localhost no
```

### Can’t connect to 5901

- Confirm TigerVNC is running:

```bash
vncserver -list
```

- Confirm the port is listening:

```bash
sudo ss -tulpn | grep :5901
```

- Confirm firewall:

```bash
sudo ufw status
sudo ufw allow 5901/tcp
```

### TigerVNC starts, but you want a different resolution

Stop it and restart with a different geometry:

```bash
vncserver -kill :1
vncserver :1 -geometry 1920x1080 -localhost no
```

### systemd user service doesn’t start on boot

1) Enable lingering:

```bash
sudo loginctl enable-linger $USER
```

2) Check service status:

```bash
systemctl --user status vncserver@:1.service --no-pager
journalctl --user -u vncserver@:1.service -n 200 --no-pager
```

### Security: Avoid exposing VNC to the internet

Best practice is to keep VNC bound to localhost and use an SSH tunnel:

On the Jetson, start VNC with localhost only:

```bash
vncserver :1 -geometry 2560x1440 -localhost yes
```

On your Mac, tunnel it:

```bash
ssh -L 5901:localhost:5901 <username>@<jetson-ip>
```

Then connect your VNC client to:
- `localhost:5901`

---

## Diagnostics (useful output to paste when asking for help)

```bash
uname -a
lsb_release -a || cat /etc/os-release
hostname -I
sudo systemctl status ssh --no-pager
vncserver -list
systemctl --user status vncserver@:1.service --no-pager
sudo ss -tulpn | egrep ':22|:5901' || true
sudo nvpmodel -q --verbose
aplay -l
arecord -l
```
