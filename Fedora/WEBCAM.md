# Getting the FaceTime HD Webcam Working on Fedora (2017 MacBook Air)

The built-in webcam on this MacBook Air is a Broadcom **FaceTime HD** camera
connected over **PCIe** (not USB), so the standard `uvcvideo` driver can't see
it. It needs the community `facetimehd` driver plus a firmware blob, and on
Fedora you also need to make sure that firmware gets baked into the initramfs.

Tested on: Fedora 44, kernel `7.0.14-201.fc44.x86_64`, 2017 MacBook Air
(`MacBookAir7,2` chassis), Broadcom `14e4:1570` camera, no Secure Boot.

## 1. Confirm you have this camera

```bash
lspci -nn | grep -i camera
```

You're looking for something like:

```
02:00.0 Multimedia controller [0480]: Broadcom Inc. and subsidiaries 720p FaceTime HD Camera [14e4:1570]
```

If you see the `14e4:1570` ID, this guide applies to you.

## 2. Check Secure Boot status

```bash
mokutil --sb-state
```

- If it says Secure Boot is **disabled/unsupported**, you're fine — skip MOK
  signing entirely.
- If it's **enabled**, the DKMS-built kernel module won't load unless you
  either disable Secure Boot in firmware settings, or sign the module and
  enroll it via MOK (not covered in this guide — RPM Fusion has docs on this).

## 3. Install build tools and enable the COPR repo

```bash
sudo dnf copr enable frgt10/facetimehd-dkms
sudo dnf install facetimehd
```

This installs the `facetimehd` DKMS package, which builds and installs the
kernel module for your current (and future) kernels automatically.

Check it built successfully:

```bash
dkms status
# should show: facetimehd/<version>, <your-kernel>, x86_64: installed
```

## 4. Extract and install the firmware

The camera needs a firmware blob originally shipped inside macOS. This script
downloads it and extracts it for you — no need to have a Mac or macOS install
around.

```bash
git clone https://github.com/patjak/facetimehd-firmware.git
cd facetimehd-firmware
make
sudo make install (important to run as root)
```

This installs the firmware to `/usr/lib/firmware/facetimehd/firmware.bin`.

## 5. Make sure the firmware is included in the initramfs (Fedora-specific)

This is the step that's easy to miss and isn't mentioned in the upstream
driver docs, but is required on Fedora. The `facetimehd` module tries to load
its firmware very early at boot — early enough that it needs to already be
inside the initramfs, not just present on disk.

Create a dracut config telling it to include the firmware file:

```bash
sudo tee /etc/dracut.conf.d/facetimehd.conf << 'EOF'
install_items+="/usr/lib/firmware/facetimehd/firmware.bin"
EOF
```

Rebuild the initramfs:

```bash
sudo dracut -f
```

Verify the firmware actually made it into the image:

```bash
sudo lsinitrd | grep -i firmware.bin
# should show: usr/lib/firmware/facetimehd/firmware.bin (~1.4 MB)
```

If this comes back empty, re-run `sudo dracut -f -v` and check the log for
errors, and double check the contents of
`/etc/dracut.conf.d/facetimehd.conf`.

## 6. Reboot and test

```bash
sudo reboot
```

After rebooting, check that the driver loaded and found the firmware:

```bash
sudo dmesg | grep -i facetimehd
```

You should **not** see `Direct firmware load for facetimehd/firmware.bin
failed`. Instead you should see lines indicating the camera registered
successfully, and:

```bash
ls -la /dev/video*
```

should show a `/dev/videoX` device.

Then just open **Cheese**, **Chromium**, or any camera app and turn the
camera on.

## Troubleshooting notes

- **`dmesg: read kernel buffer failed: Operation not permitted`** — just run
  `dmesg` with `sudo`.
- **`Direct firmware load ... failed with error -2`** in `dmesg` — the
  firmware exists on disk but isn't in the initramfs. Redo step 5.
- **`sudo echo ... >> /etc/...` gives "Permission denied"** — this is the
  classic `sudo echo` trap. `sudo` only applies to `echo`; the `>>`
  redirection runs in your normal unprivileged shell, which can't write to
  `/etc`. Use `sudo tee` instead, e.g.:
  ```bash
  echo 'some line' | sudo tee -a /etc/some/file
  ```
- **Colors look a bit off** — cosmetic issue with this reverse-engineered
  driver. There's an optional sensor-calibration-file extraction step in the
  [upstream wiki](https://github.com/patjak/facetimehd/wiki/Get-Started) if
  you want to fine-tune it later; not required for the camera to work.
- **Works in some apps but not others** — some apps (occasionally Firefox)
  are pickier about this driver via V4L2. Chromium and Cheese tend to work
  reliably.
- **After a kernel update** — DKMS should rebuild the module automatically.
  If the camera stops working after `dnf update`, check `dkms status` first.

## References

- Driver: https://github.com/patjak/facetimehd
- Firmware extractor: https://github.com/patjak/facetimehd-firmware
- COPR (DKMS package): `frgt10/facetimehd-dkms`
