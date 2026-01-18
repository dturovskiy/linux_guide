# Chrome Middle-Click Autoscroll (Ubuntu GNOME / Wayland) — Personal Setup Guide

Goal: Enable “Windows-style” autoscroll in **Google Chrome**:
- Middle-click (mouse wheel click) shows an autoscroll indicator
- Moving the mouse up/down scrolls automatically

This guide is optimized for quick re-setup after reinstall.

> Note: This approach uses an **unsupported Chrome command-line feature flag**. Chrome will show a startup warning banner about an unsupported flag. This guide intentionally **keeps** that warning (no suppression).

---

## 0) What you get (and trade-offs)

### You get
- Middle-click autoscroll behavior inside Chrome pages.

### Trade-offs
- You may want to disable GNOME “middle-click paste” (PRIMARY selection) to avoid conflicts.
- The Chrome flag may break/rename in future Chrome versions (rare, but possible).
- A startup warning banner will appear because the flag is unsupported.

---

## 1) Disable middle-click paste in GNOME (recommended)

This reduces conflicts where middle-click is used for “PRIMARY selection paste” instead of autoscroll.

Disable:

```bash
gsettings set org.gnome.desktop.interface gtk-enable-primary-paste false
```

Re-enable later (if you want default Linux behavior back):

```bash
gsettings set org.gnome.desktop.interface gtk-enable-primary-paste true
```

---

## 2) One-time test (run Chrome from Terminal)

Close Chrome completely, then run:

```bash
google-chrome --enable-blink-features=MiddleClickAutoscroll
```

Test:

1. Open a long page.
2. Middle-click on an empty area (not a link/button).
3. An autoscroll indicator should appear.
4. Move mouse up/down to scroll.

If this works, proceed to “make it permanent”.

---

## 3) Make it permanent (local .desktop launcher)

Do **not** edit files under `/usr/share/applications/` (package updates may overwrite them).
Instead, create a user-local launcher.

### 3.1 Copy the system launcher into your profile

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/google-chrome.desktop ~/.local/share/applications/
```

### 3.2 Edit the local launcher

```bash
nano ~/.local/share/applications/google-chrome.desktop
```

Find **every** `Exec=` line that launches Chrome (common examples include `%U`, `%u`, or no args) and insert the flag right after the binary:

Example:

**Before**

```text
Exec=/usr/bin/google-chrome-stable %U
```

**After**

```text
Exec=/usr/bin/google-chrome-stable --enable-blink-features=MiddleClickAutoscroll %U
```

Do this for all relevant `Exec=` entries (regular window, incognito, etc.) so all launch modes behave consistently.

### 3.3 Refresh desktop database (optional but recommended)

```bash
update-desktop-database ~/.local/share/applications
```

Then log out/in, or just relaunch Chrome from the app menu.

---

## 4) Quick verification checklist

### 4.1 Confirm you are launching the local .desktop file

Run:

```bash
grep -n '^Exec=' ~/.local/share/applications/google-chrome.desktop
```

Ensure the flag is present on the Exec lines.

### 4.2 If autoscroll doesn’t trigger

* Middle-click on a non-interactive part of the page (not a link, not a button).
* Confirm GNOME middle-click paste is disabled (Section 1).
* Fully close Chrome and reopen (Chrome sometimes keeps background processes).

---

## 5) Maintenance / after Chrome updates

### 5.1 Will updates remove your setup?

* The local launcher at `~/.local/share/applications/...` usually **survives updates**.

### 5.2 Can the feature stop working after updates?

Yes. The flag is unsupported and may be changed/removed in future versions.
If autoscroll stops working:

1. Try launching from terminal again (Section 2).
2. If the flag no longer works, consider a Chrome extension that provides autoscroll behavior.

---

## 6) Uninstall / revert

### 6.1 Restore GNOME default middle-click paste

```bash
gsettings set org.gnome.desktop.interface gtk-enable-primary-paste true
```

### 6.2 Remove your local launcher override

```bash
rm ~/.local/share/applications/google-chrome.desktop
update-desktop-database ~/.local/share/applications
```

After that, your system will use the default Chrome launcher again.

---

## 7) Notes

* This guide focuses on Chrome only.
* It’s safe to keep the startup warning banner; it simply informs you that the launch flag is unsupported.

