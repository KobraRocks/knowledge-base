# Use an Android Phone as a Webcam on Arch Linux (via `scrcpy` + `v4l2loopback`)

This guide turns your Android phone into a plug‑and‑play webcam on Arch Linux. It documents a working setup validated on an FP3 (Android 13), but it’s generally applicable to most Android devices.

---

## TL;DR (Working Command)

Once everything below is installed and configured, connect the phone over USB (USB debugging ON), then run:

```bash
scrcpy --video-source=camera \
  --camera-id=1 \
  --camera-ar=16:9 -m2560 \
  --camera-fps=30 \
  --video-codec=h264 --video-bit-rate=6M \
  --v4l2-sink=/dev/video10 \
  --no-audio --no-control
```

Select **“PhoneWebcam (/dev/video10)”** in your video app.

---

## 1) Install Packages

```bash
sudo pacman -S scrcpy android-tools v4l2loopback-dkms dkms v4l-utils android-udev
```

> `android-udev` provides udev rules so ADB works without `sudo`.

## 2) Ensure Kernel Headers (DKMS needs them)

Find your running kernel:

```bash
uname -r
```

Install matching headers (examples):

```bash
# Stock Arch kernel
sudo pacman -S linux-headers

# Or the variant you actually run
# sudo pacman -S linux-zen-headers
# sudo pacman -S linux-lts-headers
# sudo pacman -S linux-hardened-headers
```

Rebuild DKMS modules:

```bash
sudo dkms autoinstall
```

## 3) Load `v4l2loopback` and Verify

Create a virtual camera at `/dev/video10` with a nice label:

```bash
sudo modprobe v4l2loopback devices=1 video_nr=10 exclusive_caps=1 card_label="PhoneWebcam"
```

Verify:

```bash
v4l2-ctl --list-devices
# Expect something like:
# PhoneWebcam (platform:v4l2loopback-010):
# 	/dev/video10
```

## 4) Enable ADB + Authorize the Phone

On the **phone**: Enable **Developer options → USB debugging**. When plugged in, accept the **RSA fingerprint** prompt. Optional: set USB mode to **File transfer (MTP)** if required by your device.

On **Arch** (first time):

```bash
sudo usermod -aG adbusers "$USER"
sudo udevadm control --reload-rules && sudo udevadm trigger
# Re‑login or unplug/replug the phone
adb kill-server && adb start-server
adb devices  # should show ...  device
```

## 5) Start Streaming (Manual Test)

Use the validated command (rear cam often `--camera-id=1`, varies by device):

```bash
scrcpy --video-source=camera \
  --camera-id=1 \
  --camera-ar=16:9 -m2560 \
  --camera-fps=30 \
  --video-codec=h264 --video-bit-rate=6M \
  --v4l2-sink=/dev/video10 \
  --no-audio --no-control
```

Open your meeting app (Zoom/Meet/Teams/Jitsi/etc.) and choose **PhoneWebcam**.

## 6) Make the Virtual Camera Persistent Across Reboots

**Auto‑load the module at boot**

```bash
sudo tee /etc/modules-load.d/v4l2loopback.conf >/dev/null <<'EOF'
v4l2loopback
EOF

sudo tee /etc/modprobe.d/v4l2loopback.conf >/dev/null <<'EOF'
options v4l2loopback devices=1 video_nr=10 exclusive_caps=1 card_label="PhoneWebcam"
EOF
```

Test reload:

```bash
sudo modprobe -r v4l2loopback && sudo modprobe v4l2loopback
v4l2-ctl --list-devices
```

## 7) Convenience Script (`phonecam.sh`)

Create a small launcher:

```bash
sudo tee /usr/local/bin/phonecam.sh >/dev/null <<'EOF'
#!/bin/bash
set -euo pipefail

adb start-server >/dev/null || true

echo "Waiting for device (authorize on phone if prompted)…"
adb wait-for-device

if ! adb devices | awk 'NR>1{print $2}' | grep -qx device; then
  echo "❌ ADB device not authorized (check phone screen)." >&2
  exit 1
fi

# Load v4l2loopback if not present
if ! lsmod | grep -q '^v4l2loopback'; then
  sudo modprobe v4l2loopback devices=1 video_nr=10 exclusive_caps=1 card_label="PhoneWebcam"
fi

exec scrcpy --video-source=camera \
  --camera-id=1 \
  --camera-ar=16:9 -m2560 \
  --camera-fps=30 \
  --video-codec=h264 --video-bit-rate=6M \
  --v4l2-sink=/dev/video10 \
  --no-audio --no-control
EOF

sudo chmod +x /usr/local/bin/phonecam.sh
```

Usage:

```bash
phonecam.sh
```

(Optional) Add a desktop launcher `~/.local/share/applications/phonecam.desktop`:

```ini
[Desktop Entry]
Name=PhoneCam
Exec=/usr/local/bin/phonecam.sh
Icon=camera-video
Type=Application
Categories=Utility;
Terminal=true
```

## 8) (Optional) Auto‑start on Phone Plug‑in via udev

Find your phone vendor id:

```bash
lsusb | grep -i -e fairphone -e android -e your_brand
# Example for FP3 vendor id: 2ae5
```

Create a udev rule (replace `2ae5` with your vendor id):

```bash
sudo tee /etc/udev/rules.d/99-phonecam.rules >/dev/null <<'EOF'
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="2ae5", RUN+="/usr/local/bin/phonecam.sh"
EOF

sudo udevadm control --reload-rules && sudo udevadm trigger
```

> **Note:** This runs the script as root via udev. If you prefer user‑level startup, use a systemd **user** service tied to `adb` readiness instead.

---

## Troubleshooting

**Module not found / DKMS errors**
Install the **matching kernel headers** (see step 2), then:

```bash
sudo dkms autoinstall
sudo modprobe v4l2loopback
```

**`scrcpy`: "Camera video source: control disabled / Cannot request to stay awake"**
Do **not** pass `--stay-awake` in camera mode. Use `--no-control` instead.

**ADB says `unauthorized`**
Check the phone for the RSA prompt, ensure USB debugging is enabled, and that `android-udev` is installed. Revoke authorizations on the phone if needed and reconnect.

**MediaCodec / CameraAccessException / camera error state**
Close any camera apps on the phone, reboot the phone, and start with conservative settings (e.g., `-m1280 --camera-ar=16:9 --camera-fps=30`). Try the other lens (`--camera-id=0/1`) or `--camera-facing=back`. You can also try HEVC (`--video-codec=h265`) or pin a specific encoder (see encoders via `adb shell "dumpsys media.codec"`).

**Permission denied on `/dev/video10`**
Add your user to the `video` group and re‑login:

```bash
sudo usermod -aG video "$USER"
```

**App doesn’t see the camera**
Verify with `v4l2-ctl --list-devices`. In Wayland sessions, some apps require Flatpak permission tweaks or PipeWire access—test in a second app (e.g., `mpv /dev/video10` or web‑based meet).

---

## Uninstall / Revert

```bash
sudo rm -f /etc/modules-load.d/v4l2loopback.conf /etc/modprobe.d/v4l2loopback.conf /etc/udev/rules.d/99-phonecam.rules /usr/local/bin/phonecam.sh
sudo modprobe -r v4l2loopback || true
```

---

## Notes

* The exact `--camera-id` may differ across devices; list cameras with:

  ```bash
  adb shell "cmd media.camera list-cameras"
  ```
* If you update the kernel, DKMS will rebuild automatically; if not, rerun `sudo dkms autoinstall`.

---

**Enjoy your high‑quality Android webcam on Arch!** 📷
