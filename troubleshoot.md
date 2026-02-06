# Troubleshooting — Jetson Orin Nano Clean Setup (JetPack 6.2.2)

This file covers common problems when setting up **SSH**, **Chromium**, **XFCE**, and **VNC (x11vnc)**.

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

- Ensure your Mac and Jetson are on the same network.

### "Permission denied (publickey)" / password issues

- Use verbose mode to see what key is being offered:

```bash
ssh -v <username>@<ip-address>
```

- Copy your key to the Jetson:

```bash
ssh-copy-id <username>@<ip-address>
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

### Performance mode doesn’t persist

Re-apply after reboot:

```bash
sudo nvpmodel -m 2
sudo nvpmodel -q --verbose
```

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

### Snap is broken or stuck

Try repairing snapd:

```bash
sudo snap remove chromium || true
sudo snap remove snapd || true
sudo apt -y install snapd
sudo reboot
```

Then repeat the snapd downgrade steps from the README.

---

## XFCE / Login Issues

### XFCE option not visible at login

Ensure LightDM is selected:

```bash
sudo dpkg-reconfigure lightdm
sudo reboot
```

### Black screen after switching display managers

Reconnect a monitor temporarily and check LightDM:

```bash
systemctl status lightdm --no-pager
```

If LightDM isn’t running:

```bash
sudo systemctl enable --now lightdm
sudo reboot
```

---

## VNC (x11vnc) Issues

### VNC connects but shows a black screen

- x11vnc mirrors the physical desktop session `:0`. Make sure a user is logged in.
- Confirm LightDM auth path is correct in the service:
  - `-display :0`
  - `-auth /var/run/lightdm/root/:0`

Check log:

```bash
sudo tail -n 200 /var/log/x11vnc.log
```

### x11vnc service won’t start

```bash
sudo systemctl status x11vnc --no-pager
sudo journalctl -u x11vnc -n 200 --no-pager
```

Common causes:
- Wrong username in `-rfbauth /home/<YOUR_USERNAME>/.vnc/passwd`
- Password file missing (re-run):

```bash
sudo x11vnc -storepasswd
```

### Port 5900 not reachable

- Verify it’s listening:

```bash
sudo ss -tulpn | grep :5900
```

- Verify firewall:

```bash
sudo ufw status
sudo ufw allow 5900/tcp
```

---

## Timeshift Snapshot Issues

### Timeshift UI won’t open (headless)

CLI still works:

```bash
sudo timeshift --create --comments "Base-Clean-Setup"
sudo timeshift --list
```

---

## Diagnostics (useful output to paste when asking for help)

```bash
uname -a
lsb_release -a || cat /etc/os-release
hostname -I
sudo systemctl status ssh --no-pager
sudo systemctl status x11vnc --no-pager
sudo journalctl -u x11vnc -n 80 --no-pager
sudo nvpmodel -q --verbose
aplay -l
arecord -l
```
