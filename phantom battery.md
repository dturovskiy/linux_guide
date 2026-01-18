# Hide Phantom Battery Icon (GNOME / Wayland) — No Extensions, Persistent

Goal: Remove the battery icon from the GNOME top bar and the battery entry from the Quick Settings menu **when the laptop has no physical battery**, but the system still exposes a phantom `BAT0` via ACPI/UPower (often shown as 0% / red).

This method is **extension-free** and survives reboots by unbinding the ACPI battery device on every boot.

---

## 1) Confirm the symptom (phantom BAT0 exists)

Check UPower devices:

```bash
upower -e
````

If you see something like:

* `/org/freedesktop/UPower/devices/battery_BAT0`

Also confirm kernel power supply devices:

```bash
ls -1 /sys/class/power_supply
```

If you see `BAT0`, GNOME will usually show a battery indicator.

---

## 2) Identify the ACPI battery device (one-time)

On many machines the phantom battery is exposed as the ACPI device `PNP0C0A:00`.

You can confirm the driver binding:

```bash
realpath /sys/class/power_supply/BAT0/device
realpath /sys/class/power_supply/BAT0/device/driver
cat /sys/class/power_supply/BAT0/device/modalias 2>/dev/null || true
```

Common modalias:

```text
acpi:PNP0C0A:
```

---

## 3) Quick one-time test (temporary)

This removes the battery immediately (until reboot):

```bash
echo "PNP0C0A:00" | sudo tee /sys/bus/acpi/drivers/battery/unbind
```

Verify:

```bash
ls -1 /sys/class/power_supply
upower -e
```

Expected: `BAT0` and `battery_BAT0` disappear, and GNOME removes the icon immediately.

> Note: After reboot, `BAT0` will return unless you make it persistent (next section).

---

## 4) Make it persistent (recommended): systemd oneshot service

### 4.1 Create the service

```bash
sudo tee /etc/systemd/system/hide-phantom-battery.service >/dev/null <<'EOF'
[Unit]
Description=Hide phantom ACPI battery (unbind PNP0C0A:00)
After=systemd-modules-load.service
Before=display-manager.service
ConditionPathExists=/sys/bus/acpi/drivers/battery/unbind

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo PNP0C0A:00 > /sys/bus/acpi/drivers/battery/unbind'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

### 4.2 Enable it (and run it now)

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hide-phantom-battery.service
```

### 4.3 Verify immediately

```bash
ls -1 /sys/class/power_supply
upower -e
```

Expected: only `AC` remains, no `BAT0`.

### 4.4 Verify persistence after reboot

```bash
sudo reboot
```

After reboot:

```bash
ls -1 /sys/class/power_supply
upower -e
```

Expected: still no `BAT0`, and the battery icon stays gone.

---

## Why this approach

* GNOME shows the battery indicator when UPower reports a battery device.
* The system can expose a phantom battery via ACPI even if no physical battery is present.
* Blacklisting the kernel `battery` driver is not always reliable (e.g., when built into the kernel or loaded early).
* A systemd boot-time unbind is deterministic, reversible, and does not require GNOME extensions.

---

## Rollback / Undo

Disable and remove the service:

```bash
sudo systemctl disable --now hide-phantom-battery.service
sudo rm /etc/systemd/system/hide-phantom-battery.service
sudo systemctl daemon-reload
sudo reboot
```

After reboot, `BAT0` should reappear (if firmware exposes it), and GNOME may show the icon again.

---

## Notes

* If your ACPI battery device is not `PNP0C0A:00`, adjust the service to echo the correct device name.
  You can list currently bound devices under:

```bash
ls -1 /sys/bus/acpi/drivers/battery/
```

Look for entries similar to `PNP0C0A:00`.
