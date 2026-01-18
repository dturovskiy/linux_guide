# Fix: Brightness Slider Changes Values but Screen Brightness Does Not Change (Linux)

This guide targets a common Linux laptop issue where:

- The desktop brightness slider/keys move normally, and
- `/sys/class/backlight/*/brightness` changes,
- but the **actual panel brightness does not change**.

This is usually caused by Linux selecting a backlight interface (e.g., `intel_backlight`) that updates sysfs values but does not control the real hardware on a particular BIOS/EC/panel combination.

The fix below forces the kernel to expose and prefer the **ACPI Video** backlight interface (`acpi_video0`, `acpi_video1`), which on many affected machines is the path that truly controls panel brightness.

This is **not specific to any single laptop brand**. It can appear across vendors and models.

> Works on both **Wayland and X11** because this is a **kernel/ACPI selection issue**, not a display server issue.

---

## Symptoms

- Brightness slider/keys move, but the screen brightness stays the same.
- Writing to `/sys/class/backlight/<device>/brightness` changes numbers, but the panel does not dim/brighten.
- You may see only one backlight device (commonly `intel_backlight`) or the “wrong” one is being used.

---

## Quick Diagnosis

List available backlight devices:

```bash
ls -1 /sys/class/backlight
```

If you only see something like `intel_backlight`, try changing it manually:

```bash
d="/sys/class/backlight/intel_backlight"
max="$(cat "$d/max_brightness")"

echo 1 | sudo tee "$d/brightness" >/dev/null
sleep 1
echo "$max" | sudo tee "$d/brightness" >/dev/null
```

If values change but the panel brightness **does not**, proceed with the fix.

---

## Fix (Recommended): Force ACPI Video Backlight

### What to change (universal)

Add this kernel parameter:

```text
acpi_backlight=video
```

How you apply it depends on your bootloader (GRUB, systemd-boot, rEFInd, etc.).

---

## Apply the Fix

### Option A: GRUB (most common)

#### 1) Edit GRUB kernel command line

```bash
sudo nano /etc/default/grub
```

Find:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

Append the parameter inside the quotes:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_backlight=video"
```

Keep any existing parameters; just add `acpi_backlight=video`.

#### 2) Rebuild GRUB config

* Debian/Ubuntu and many derivatives:

```bash
sudo update-grub
```

* Other distributions:

  * Rebuilding the GRUB configuration varies by distro and by whether you use UEFI or legacy BIOS.
  * Use your distribution’s official documentation to regenerate GRUB’s configuration file.

#### 3) Reboot

```bash
sudo reboot
```

---

### Option B: systemd-boot (common on some setups)

1. Identify the loader entry you boot (typically under `/boot/loader/entries/`):

```bash
ls -1 /boot/loader/entries
```

2. Edit the chosen entry file and append `acpi_backlight=video` to the `options` line. Example:

```text
options quiet splash acpi_backlight=video
```

3. Reboot:

```bash
sudo reboot
```

> Exact paths may differ (`/boot` vs `/efi`) depending on your layout.

---

## Verify

Confirm the kernel parameter is active:

```bash
cat /proc/cmdline | tr ' ' '\n' | grep '^acpi_backlight='
```

Expected output:

```text
acpi_backlight=video
```

Confirm ACPI backlight devices exist:

```bash
ls -1 /sys/class/backlight
```

Expected to include one or more of:

* `acpi_video0`
* `acpi_video1`

At this point, desktop brightness controls typically start working.

---

## Optional: Determine Which ACPI Device Actually Controls Brightness

Some systems expose multiple ACPI video devices; only one may be effective.

```bash
for dev in acpi_video0 acpi_video1; do
  d="/sys/class/backlight/$dev"
  [ -d "$d" ] || continue

  echo "==== $dev ===="
  max="$(cat "$d/max_brightness")"
  low=$((max/10)); [ "$low" -lt 1 ] && low=1

  echo "set low=$low"
  echo "$low" | sudo tee "$d/brightness" >/dev/null
  sleep 1

  echo "set max=$max"
  echo "$max" | sudo tee "$d/brightness" >/dev/null
  sleep 1
done
```

Use your eyes: whichever device dims/brightens the panel is the real one.

---

## Cleanup: Remove Experimental i915 Backlight Overrides (If You Tried Them)

If you previously created `i915` module overrides (e.g., `enable_dpcd_backlight=...`) in `/etc/modprobe.d/`, it is recommended to disable them to avoid conflicts.

Find overrides:

```bash
grep -R --line-number -E '^\s*options\s+i915\b' /etc/modprobe.d /usr/lib/modprobe.d /run/modprobe.d 2>/dev/null || true
```

To disable a custom config file safely (example):

```bash
sudo mv /etc/modprobe.d/<your-i915-file>.conf /etc/modprobe.d/<your-i915-file>.conf.disabled
```

### If your system loads i915 early, you may need to rebuild the initramfs

* Debian/Ubuntu:

```bash
sudo update-initramfs -u -k all
sudo reboot
```

* Arch (mkinitcpio):

```bash
sudo mkinitcpio -P
sudo reboot
```

* Fedora/RHEL (dracut):

```bash
sudo dracut -f
sudo reboot
```

After reboot, you can check whether i915 is forced (optional):

```bash
sudo cat /sys/module/i915/parameters/enable_dpcd_backlight 2>/dev/null || true
```

Typical default is `-1` (auto).

---

## If This Fix Does Not Work

Firmware implementations vary. Try one of the following **one at a time** (do not stack multiple backlight parameters initially), rebooting after each change:

* `acpi_backlight=native`
* `acpi_backlight=vendor`
* `acpi_backlight=none`

You can also try:

* `video.use_native_backlight=1`

Testing alternatives:

1. Apply a single alternative parameter in your bootloader config
2. Reboot
3. Re-check `/sys/class/backlight` and real brightness behavior

---

## Rollback (Restore Default Behavior)

Remove the added parameter from your bootloader configuration (e.g., remove `acpi_backlight=video` from GRUB or systemd-boot entry), rebuild bootloader config if needed, and reboot.

For GRUB on Debian/Ubuntu:

```bash
sudo update-grub
sudo reboot
```

---

## Notes / Scope

* Not tied to any single vendor. The root cause is typically BIOS/EC/backlight routing and kernel interface selection.
* The “best” kernel parameter may differ by machine, but the methodology (switch kernel backlight interface, verify in `/sys/class/backlight`) is broadly applicable.
* Examples intentionally avoid machine-specific IDs; use the commands shown to inspect your own system.
