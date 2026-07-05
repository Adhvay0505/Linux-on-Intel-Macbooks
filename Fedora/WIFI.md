## Getting the wifi drivers installed and running

By default, on an intel macbook air, WiFi will not work
To get Wi-Fi working on an Intel MacBook, you need to install the proprietary Broadcom wireless drivers. Since your Mac's Wi-Fi card is offline, establish a temporary internet connection using a USB-Ethernet adapter or by connecting your smartphone via a USB cable and enabling "USB Tethering/Hotspot

1. Enable RPM Fusion Repositories

```bash 
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
```

```bash
sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

2. Update Your System and Install Drivers
Update your package cache to ensure full compatibility with your current kernel, then install the Broadcom driver packages (broadcom-wl)

```bash
sudo dnf update --refresh
```

```bash
sudo dnf install broadcom-wl
```

3. Reboot
