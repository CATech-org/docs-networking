# Fixing WiFi Drops Caused by USB Autosuspend on Ubuntu

A USB WiFi adapter can silently go to sleep after a period of inactivity, killing the connection and with it, remote/VPN access to the machine.

---

## Symptom

- WiFi connection drops after roughly 30 minutes of use, with no obvious trigger.
- If the machine is remote-accessed over a VPN (e.g. WireGuard, see [1-wireguard-vpn-setup.md](1-wireguard-vpn-setup.md)), the VPN tunnel goes down with it. The machine becomes unreachable from outside, not just off the VPN.
- Reconnecting WiFi manually (or a reboot) restores the connection, only for it to drop again after a similar interval.

> [!important] This looks like a driver bug, a router issue, or a flaky adapter, but none of these is the cause. The actual cause is a power-management feature silently suspending the USB device the WiFi adapter is connected through. It is easy to chase the wrong fix (driver reinstalls, router settings) before finding this.

---

## 1. Root Cause

Linux's USB subsystem has **autosuspend**: after a period of inactivity, the kernel powers down idle USB devices to save energy, then wakes them on the next access. This works fine for a mouse or a USB drive. For a USB WiFi adapter, "inactivity" can be misjudged. The adapter gets suspended while the connection is still logically active, and it does not reliably wake back up, so the interface just disappears.

This is distinct from WiFi power management (`iw dev wlan0 get power_save`), which is a separate, adapter-level setting. The fix here applies to the USB port/device power state, not the WiFi radio's own power saving.

---

## 2. Diagnosis

Identify the WiFi adapter's USB device and check its current autosuspend delay.

```bash
# Find the adapter's USB bus/device ID
lsusb

# Find its path under /sys and check the current autosuspend delay (in ms)
for d in /sys/bus/usb/devices/*/power/autosuspend_delay_ms; do
  echo "$d: $(cat "$d")"
done
```

A positive number is how many milliseconds of idle time before the kernel suspends the device. `-1` means autosuspend is disabled for it. That is the value the fix sets.

> [!note] Two related but different files
> `power/control` (`auto` / `on`) is a coarser on/off switch for whether autosuspend is allowed at all. `power/autosuspend_delay_ms` (or the older `power/autosuspend`, in seconds) is the actual delay value, where `-1` is the "never suspend" sentinel. Setting the delay to `-1` is the more direct and commonly used fix for this symptom.

> [!tip] Confirming the cause before fixing it
> Watch `dmesg -w` or `journalctl -f` while the adapter is idle. A `usb ... USB disconnect` or similar power-state message appearing around the time the connection drops confirms this is autosuspend, not something upstream (router, driver, DHCP lease).

---

## 3. The Fix: Disable USB Autosuspend for the Adapter

### Test first (temporary, resets on reboot)

Before making anything permanent, confirm the fix works:

```bash
# Replace with the actual sysfs path for your adapter, found above
echo 'on' | sudo tee /sys/bus/usb/devices/<device-id>/power/control
```

Leave the machine idle past the point it previously dropped (e.g. 30+ minutes) and confirm the connection holds.

> [!tip] A small script that sets `power/control` to `on` for the relevant device and then sleeps and monitors is a reasonable way to validate this before writing a permanent rule.

### Make it permanent

Since `/sys` settings do not survive a reboot and device paths are not guaranteed stable across boots, the reliable way to persist this is a **udev rule** matching the device by vendor/product ID rather than by its current sysfs path.

1. Get the vendor and product ID from `lsusb` (format `idVendor:idProduct`, e.g. `0bda:8153`).
2. Create a udev rule:

```bash
sudo tee /etc/udev/rules.d/99-usb-wifi-no-autosuspend.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="<vendor-id>", ATTR{idProduct}=="<product-id>", TEST=="power/control", ATTR{power/control}="on"
EOF
```

3. Reload udev rules and re-trigger:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

4. Verify it stuck:

```bash
cat /sys/bus/usb/devices/<device-id>/power/control   # should read "on"
```

> [!important] A udev rule keyed to `idVendor`/`idProduct` survives reboots and re-enumeration, unlike a one-off `echo` to `/sys` which is wiped every time the device is re-plugged or the machine restarts. This is what makes the fix permanent.

---

## Quick Reference

|Command|Purpose|
|---|---|
|`lsusb`|List USB devices, find vendor/product ID|
|`cat /sys/bus/usb/devices/<id>/power/control`|Check current autosuspend state (`auto` vs `on`)|
|`echo 'on' \| sudo tee .../power/control`|Temporarily disable autosuspend (test only, resets on reboot)|
|udev rule in `/etc/udev/rules.d/`|Persist the fix across reboots, keyed by vendor/product ID|
|`sudo udevadm control --reload-rules && sudo udevadm trigger`|Apply a new or changed udev rule without rebooting|

---

## Notes

- This is specific to **USB** WiFi adapters. A PCIe or built-in WiFi card does not go through this code path and needs a different fix if it exhibits similar symptoms (usually WiFi power management itself, `iwconfig wlan0 power off` or the NetworkManager equivalent).
- Worth checking on any machine that stays reachable 24/7 over a VPN. A WiFi drop here does not just lose local connectivity, it silently takes down remote access too, which is a lot less obvious to notice than a wired connection dropping.
