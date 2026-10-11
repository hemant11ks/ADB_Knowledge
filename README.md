# Android Debug Bridge (ADB) — Complete Practical Guide

> A GitHub-ready reference for Android developers, testers, DevOps engineers, and AOSP/automotive engineers.

**ADB (Android Debug Bridge)** is a command-line tool included with the Android SDK Platform-Tools. It lets a computer communicate with an Android phone, tablet, emulator, Android TV, or supported automotive test target to install applications, execute shell commands, transfer files, collect logs, and troubleshoot devices.

**Contents:** [Architecture](#1-architecture) · [Installation](#2-installation) · [Connect](#3-connect-a-device) · [Commands](#4-essential-adb-commands) · [Applications](#5-application-management) · [Files](#6-file-transfer) · [Shell](#7-device-shell-and-system-information) · [Logcat](#8-logs-and-debugging) · [Screenshots](#9-screenshots-and-screen-recording) · [Wireless](#10-wireless-debugging) · [Automation](#14-automation-and-cicd) · [Troubleshooting](#16-troubleshooting) · [Cheat Sheet](#17-quick-cheat-sheet)

---

## 1. Architecture

ADB uses three components:

```text
Developer PC / CI runner                    Android device / emulator
+-------------------------+                 +-------------------------+
| adb client (CLI)        |                 | adbd daemon             |
|       |                 |    USB / Wi-Fi  |                         |
| adb server (TCP 5037)   | <-------------> | Android shell/services  |
+-------------------------+                 +-------------------------+
```

- **Client:** The `adb` command run from your computer or automation script.
- **Server:** Manages connections to devices; normally listens on **localhost TCP port 5037**.
- **Daemon (`adbd`):** Runs on the Android target and handles commands.
- **ADB vs. Fastboot:** `adb` talks to a running Android system (or supported recovery); `fastboot` operates in a compatible bootloader/fastboot mode. They are distinct tools.

## 2. Installation

### Ubuntu / Debian

Quick distribution-packaged installation:

```bash
sudo apt update
sudo apt install -y adb fastboot android-sdk-platform-tools-common
adb version
fastboot --version
```

> Package names and availability vary by distro. Ubuntu's packaged ADB can be older than Google's current Platform-Tools. For new features, prefer the official download below. Avoid mixing multiple `adb` executables in your `PATH`.

### Latest official Platform-Tools (Linux)

Download from **[Google's official SDK Platform-Tools page](https://developer.android.com/tools/releases/platform-tools)**, accept its terms, and unpack the Linux ZIP file:

```bash
unzip platform-tools-latest-linux.zip -d "$HOME/android-sdk"
# For Bash:
echo 'export PATH="$HOME/android-sdk/platform-tools:$PATH"' >> ~/.bashrc
source ~/.bashrc
# For Zsh, add the same export line to ~/.zshrc and source ~/.zshrc instead.
which adb
adb version
```

If `~/android-sdk/platform-tools` already exists, back it up or replace it intentionally rather than stacking copies.

### Windows

1. Download the **Windows** ZIP from the [official Platform-Tools page](https://developer.android.com/tools/releases/platform-tools).
2. Extract it to `C:\platform-tools`.
3. In PowerShell, run:

```powershell
cd C:\platform-tools
.\adb.exe version
.\adb.exe devices -l
```

To use `adb` in any terminal, add `C:\platform-tools` to your user `PATH` and reopen PowerShell. Some vendors require an OEM USB driver.

### macOS

Install official Platform-Tools from the link above, or (if you use Homebrew):

```bash
brew install android-platform-tools
adb version
```

### Check installation

```bash
adb version
command -v adb       # Linux/macOS
adb help
```

## 3. Connect a device

### Enable Developer Options and USB debugging

1. On the Android device: **Settings → About phone → Build number**, tap it seven times (menu names vary by vendor).
2. Open **Settings → Developer options → USB debugging** and enable it.
3. Connect a **data-capable** USB cable.
4. Approve the **Allow USB debugging?** prompt on the device. Only authorize trusted computers.

```bash
adb devices
adb devices -l
```

Example:

```text
List of devices attached
R58M12345AB  device product:example model:Pixel_Example transport_id:1
```

| Device state | Meaning | Suggested action |
|---|---|---|
| `device` | Connected and authorized | Ready to use |
| `unauthorized` | Authorization required | Unlock device and accept its RSA prompt |
| `offline` | Transport exists but unavailable | Reconnect and restart ADB server |
| No device | USB/driver/cable/permissions issue | Check cable, `lsusb`, udev rules, drivers |

### Manage the ADB server

```bash
adb start-server
adb kill-server
adb reconnect
adb devices -l
```

### Select one of multiple devices

```bash
adb devices -l
adb -s SERIAL shell getprop ro.product.model
adb -s SERIAL install app-debug.apk
adb -d shell getprop ro.build.version.release  # USB device, when unambiguous
adb -e shell getprop ro.build.version.release  # Emulator, when unambiguous
```

Or set `ANDROID_SERIAL` for a terminal session:

```bash
export ANDROID_SERIAL=SERIAL
adb shell getprop ro.product.model
unset ANDROID_SERIAL
```

## 4. Essential ADB commands

| Command | Purpose |
|---|---|
| `adb devices -l` | List devices and additional details |
| `adb shell` | Start interactive device shell |
| `adb shell COMMAND` | Run one command on the device |
| `adb reboot` | Reboot Android |
| `adb reboot bootloader` | Reboot into bootloader mode (device-dependent) |
| `adb reboot recovery` | Reboot into recovery (device-dependent) |
| `adb wait-for-device` | Wait for device transport to become available |
| `adb get-state` | Show target's state |
| `adb get-serialno` | Show device serial |
| `adb install app.apk` | Install APK |
| `adb pull REMOTE LOCAL` | Copy device file to computer |
| `adb push LOCAL REMOTE` | Copy computer file to device |
| `adb logcat` | Stream Android logs |
| `adb bugreport` | Create diagnostic bugreport |
| `adb exec-out screencap -p > screen.png` | Save screenshot locally |

> `adb wait-for-device` waits for a device connection, **not necessarily for Android to finish booting**.

## 5. Application management

```bash
# Install a single APK
adb install app-debug.apk

# Replace the installed version (subject to signing/package compatibility)
adb install -r app-debug.apk

# Grant eligible runtime permissions on install
adb install -g app-debug.apk

# List packages
adb shell pm list packages
adb shell pm list packages -3       # Third-party apps
adb shell pm list packages -s       # System apps

# Search installed package names on the host (Linux/macOS)
adb shell pm list packages | grep 'example'

# Find the installed APK path
adb shell pm path com.example.app

# Uninstall (removes app data under normal uninstall semantics)
adb uninstall com.example.app

# Clear application data — destructive
adb shell pm clear com.example.app

# Force-stop application
adb shell am force-stop com.example.app

# Launch a known activity (adjust package and activity)
adb shell am start -n com.example.app/.MainActivity
```

For split APK packages, use `adb install-multiple base.apk split_config.*.apk` with the correct compatible split set. Bundle `.aab` files are not directly installable with `adb install`; generate/install APK artifacts with an appropriate build/bundle tool.

## 6. File transfer

```bash
# Computer → device
adb push ./config.json /sdcard/Download/config.json

# Device → computer
adb pull /sdcard/Download/config.json ./config-copy.json

# Copy a folder
adb pull /sdcard/DCIM ./phone-photos

# List files on device
adb shell ls -lah /sdcard/Download/

# Remove your test file — destructive
adb shell rm /sdcard/Download/config.json
```

**Storage caveats:** `/sdcard` is usually shared/emulated storage. Access to private locations such as `/data/data/` is restricted on production builds. For debuggable apps, `run-as` may allow access to that app's private files:

```bash
adb shell run-as com.example.app ls files
```

## 7. Device shell and system information

```bash
adb shell                          # Interactive shell; exit with 'exit'
adb shell getprop ro.product.model
adb shell getprop ro.product.manufacturer
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.type
adb shell getprop ro.boot.serialno   # May be restricted/unavailable
adb shell uname -a
adb shell id
adb shell date
adb shell uptime
adb shell df -h
adb shell cat /proc/meminfo
adb shell getprop | head -n 30
```

### CPU, memory, processes, and services

```bash
adb shell top -n 1
adb shell ps -A
adb shell dumpsys meminfo com.example.app
adb shell dumpsys cpuinfo
adb shell dumpsys battery
adb shell dumpsys activity activities
adb shell dumpsys package com.example.app
adb shell service list
```

### Display and input automation

```bash
adb shell wm size
adb shell wm density
adb shell input keyevent KEYCODE_HOME
adb shell input keyevent KEYCODE_BACK
adb shell input tap 500 1000
adb shell input swipe 300 1000 300 300 500
adb shell input text 'hello%sworld'  # %s represents space
```

Coordinates depend on resolution, display rotation, and UI state. Avoid hardcoding them for heterogeneous test fleets where possible.

## 8. Logs and debugging

### Logcat

```bash
adb logcat                           # Live logs
adb logcat -c                        # Clear log buffers; destructive
adb logcat -d > logcat.txt           # Dump buffered logs to host file
adb logcat -v threadtime             # Timestamp, PID/TID, priority, tag
adb logcat -b crash -d > crash.txt  # Crash buffer (where available)
adb logcat '*:E'                      # Only ERROR and higher priority
adb logcat -s ActivityManager:I '*:S'  # Filter tag, silence others
```

For a specific app, first get its PID and then filter logs (supported on modern Platform-Tools):

```bash
adb shell pidof com.example.app
adb logcat --pid=$(adb shell pidof -s com.example.app | tr -d '\r')
```

The PID changes on restart, so this command needs re-running when the app process changes. Filtering logs may also be restricted by Android permissions.

### Other diagnostics

```bash
adb bugreport ./bugreports        # Large bugreport ZIP, device-dependent
adb shell dumpsys > dumpsys.txt
adb shell dmesg                   # Often denied on production builds
adb shell settings get global airplane_mode_on
```

> Logcat and bugreports can contain identifiers, account details, tokens, locations, and other sensitive data. Review and redact files before sharing them or uploading to CI artifacts.

## 9. Screenshots and screen recording

```bash
# Capture PNG directly to the host (avoids shell newline conversion)
adb exec-out screencap -p > screenshot.png

# Alternative: store then pull
adb shell screencap -p /sdcard/screenshot.png
adb pull /sdcard/screenshot.png .

# Record a short video; stop with Ctrl+C
adb shell screenrecord /sdcard/demo.mp4
adb pull /sdcard/demo.mp4 .
```

`screenrecord` availability, audio support, maximum duration, and encoder behavior depend on Android version/device. Protected content can appear blank.

## 10. Wireless debugging

### Android 11+ pairing (recommended)

Both computer and Android device should be on the **same trusted Wi-Fi network**. On the phone, enable **Developer options → Wireless debugging → Pair device with pairing code**.

```bash
# Use the PAIRING address and port shown in the phone's pairing dialog
adb pair 192.168.1.50:37123
# Enter the pairing code when prompted.

# Use the DEVICE debugging address/port shown on the Wireless debugging screen
# (it is usually different from the pairing port).
adb connect 192.168.1.50:39001
adb devices -l
adb disconnect 192.168.1.50:39001
```

Modern ADB can discover and reconnect paired devices automatically on supported networks. If not, run `adb connect` with the current debugging port. **Do not expose ADB to public/untrusted networks.**

### Legacy TCP/IP over USB (only on trusted test networks)

```bash
adb devices
adb tcpip 5555
# Find the phone's reachable LAN IP using device settings.
adb connect 192.168.1.50:5555
adb devices
# Revert to USB debugging transport while USB is connected:
adb usb
```

This legacy transport has different security properties and is not a substitute for Android 11+ pairing. Disable it when finished.

## 11. Port forwarding and reverse forwarding

```bash
# Host localhost:8080 → device localhost:8080
adb forward tcp:8080 tcp:8080
adb forward --list
adb forward --remove tcp:8080

# Device localhost:3000 → host localhost:3000
# Useful for accessing a development backend running on your computer.
adb reverse tcp:3000 tcp:3000
adb reverse --list
adb reverse --remove tcp:3000
```

Use explicit device selection (`-s SERIAL`) if multiple targets are attached.

## 12. Reboot, recovery, and engineering builds

```bash
adb reboot
adb reboot bootloader
adb reboot recovery
adb shell getprop sys.boot_completed  # 1 usually means Android boot finished
```

The following require a compatible **userdebug/eng** build or suitable elevated access and are commonly **not permitted on production `user` builds**:

```bash
adb root
adb remount
adb shell id
adb shell su -c id               # Only if 'su' exists and is authorized
```

`adb remount` can depend on verified boot, dynamic partitions, overlayfs, and device state. Never assume the system image is writable. **Unlocking bootloaders, disabling verification, flashing partitions, and `fastboot -w` can erase data or brick a target**; they are intentionally not presented as routine commands here.

## 13. Android Automotive / AOSP practical workflows

For an authorized AOSP development board or automotive head unit:

```bash
# Inventory build information
adb -s SERIAL shell getprop ro.build.fingerprint
adb -s SERIAL shell getprop ro.build.type
adb -s SERIAL shell getprop ro.hardware

# Check availability of boot-complete signal
adb -s SERIAL shell getprop sys.boot_completed

# Collect diagnostic bundle into a local directory
mkdir -p artifacts
adb -s SERIAL logcat -d -v threadtime > artifacts/logcat.txt
adb -s SERIAL shell dumpsys battery > artifacts/battery.txt
adb -s SERIAL shell getprop > artifacts/properties.txt
adb -s SERIAL exec-out screencap -p > artifacts/screen.png

# Check package and activity state
adb -s SERIAL shell pm list packages
adb -s SERIAL shell dumpsys activity activities
```

Automotive targets often have vendor-specific services, USB configurations, custom serial/network bridges, and security policies. Check the board's BSP documentation before using privileged ADB operations. `ro.hardware` and many `getprop` values are not standardized across vendors.

## 14. Automation and CI/CD

### Safe Bash smoke-test example

Create `scripts/adb-smoke-test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${ANDROID_SERIAL:?Set ANDROID_SERIAL to the intended device serial}"
APK="${1:?Usage: $0 path/to/app-debug.apk [package.name]}"
PACKAGE="${2:-}"

mkdir -p artifacts
adb -s "$ANDROID_SERIAL" wait-for-device

# Wait for Android framework readiness, not just USB transport.
ready=0
for attempt in $(seq 1 60); do
  if [[ "$(adb -s "$ANDROID_SERIAL" shell getprop sys.boot_completed | tr -d '\r')" == "1" ]]; then
    ready=1
    break
  fi
  sleep 2
done
if [[ "$ready" -ne 1 ]]; then
  echo 'Device did not complete Android boot in time' >&2
  exit 1
fi

adb -s "$ANDROID_SERIAL" install -r "$APK"
if [[ -n "$PACKAGE" ]]; then
  adb -s "$ANDROID_SERIAL" shell pm path "$PACKAGE"
fi
adb -s "$ANDROID_SERIAL" logcat -d -v threadtime > artifacts/logcat.txt
adb -s "$ANDROID_SERIAL" exec-out screencap -p > artifacts/screenshot.png
printf 'Smoke test completed on %s\n' "$ANDROID_SERIAL"
```

Run:

```bash
chmod +x scripts/adb-smoke-test.sh
ANDROID_SERIAL=SERIAL ./scripts/adb-smoke-test.sh ./app-debug.apk com.example.app
```

**This is an installation/connectivity smoke test, not a complete functional UI test.** In real CI, add application-specific assertions and always use a job-level timeout and dedicated test hardware.

### Jenkins Pipeline example

```groovy
pipeline {
    agent { label 'android-test-agent' }
    options { timeout(time: 10, unit: 'MINUTES') }
    environment {
        ANDROID_SERIAL = 'REPLACE_WITH_DEVICE_SERIAL'
    }
    stages {
        stage('Check ADB') {
            steps {
                sh 'adb version && adb devices -l'
                sh 'adb -s "$ANDROID_SERIAL" wait-for-device'
            }
        }
        stage('Install and smoke test') {
            steps {
                sh 'bash scripts/adb-smoke-test.sh app/build/outputs/apk/debug/app-debug.apk com.example.app'
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'artifacts/**', allowEmptyArchive: true
        }
    }
}
```

**CI considerations:**

- Assign each physical target to a single job at a time (device locks or dedicated executors).
- Choose targets by `-s` / `ANDROID_SERIAL` rather than assuming only one device.
- Use supported USB passthrough, reliable power, and persistent udev permissions on Linux agents.
- Restrict device access to trusted runners and review diagnostic artifacts for private data.
- Do not run destructive commands against shared test devices.

## 15. Helpful shell one-liners

```bash
# Verify Android boot complete
adb shell getprop sys.boot_completed

# Display battery details
adb shell dumpsys battery

# Display active network interfaces (device dependent)
adb shell ip addr

# Inspect disk free space
adb shell df -h /data

# Get screen resolution
adb shell wm size

# Find whether an APK is installed
adb shell pm path com.example.app

# Save system properties
adb shell getprop > device-properties.txt

# Save a quick error-log snapshot
adb logcat -d '*:E' > errors.txt
```

## 16. Troubleshooting

| Problem | Check / fix |
|---|---|
| `adb: command not found` | Install Platform-Tools; verify executable location and `PATH` |
| `unauthorized` | Unlock device, accept USB debugging authorization; revoke authorizations and reconnect if needed |
| No USB device detected | Use data cable/port; check `lsusb`, USB mode, OS drivers, device debugging settings |
| Linux `no permissions` | Install appropriate udev rules, replug device, inspect permissions; do not habitually run `sudo adb` |
| `offline` | Disconnect/reconnect; `adb kill-server && adb start-server` |
| `more than one device/emulator` | Use `adb -s SERIAL ...` or `ANDROID_SERIAL` |
| App install fails `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | Existing installation has incompatible signing key; coordinate a safe uninstall/re-signing strategy |
| `INSTALL_FAILED_VERSION_DOWNGRADE` | Use a correctly versioned APK; downgrade options depend on target/app and can impact data |
| `adbd cannot run as root in production builds` | Expected on locked-down `user` builds; use authorized userdebug/eng device |
| Wireless connection times out | Confirm current device IP/port, same network, firewall and client isolation settings |
| Device booted but app not ready | Use `sys.boot_completed` and app-specific readiness checks |
| `adb remount` fails | Verify build type, AVB/verity/overlayfs policies, and board-specific setup |

### Linux diagnostics

```bash
lsusb
adb devices -l
id -nG
ls -l /dev/bus/usb/*/* 2>/dev/null | head
which -a adb
adb kill-server && adb start-server
```

For Ubuntu, distro-provided `android-sdk-platform-tools-common` may supply udev rules. Device-vendor rules and `plugdev` group membership can vary by distribution; avoid chmodding USB devices globally.

## 17. Quick cheat sheet

```bash
adb version                         # Version
adb devices -l                     # Connected targets
adb -s SERIAL shell                # Target-specific shell
adb install -r app.apk             # Install / replace
adb uninstall com.example.app      # Remove an app
adb shell pm list packages -3      # User-installed packages
adb push local.txt /sdcard/        # Upload
adb pull /sdcard/file.txt .        # Download
adb logcat -d > logcat.txt         # Save buffered logs
adb exec-out screencap -p > x.png  # Capture screenshot
adb shell dumpsys battery          # Battery info
adb shell getprop                  # Device properties
adb reboot                         # Restart Android
adb kill-server && adb start-server # Restart local ADB server
```

## 18. Security and best practices

- Use ADB only on devices you own or are explicitly authorized to administer.
- Disable USB/wireless debugging when no longer needed, especially on personal devices.
- Avoid TCP debugging on shared/public networks and keep Platform-Tools updated.
- Double-check the device serial before rebooting, clearing app data, or deleting files.
- Be cautious with root, recovery, bootloader, remount, and flashing operations.
- Treat logs, screenshots, and bugreports as potentially sensitive artifacts.
- Favor reproducible non-root workflows in CI; isolate and reserve physical test devices.

## 19. Official references

- [Android Debug Bridge documentation](https://developer.android.com/tools/adb)
- [SDK Platform-Tools downloads and release notes](https://developer.android.com/tools/releases/platform-tools)
- [Run apps on a hardware device](https://developer.android.com/studio/run/device)
- [Android Developer tools](https://developer.android.com/tools)

---

**Maintainer note:** Replace `SERIAL`, `com.example.app`, APK paths, and sample IP addresses with values from your environment. Features and permissions differ between Android versions, vendors, and build variants.

**Last reviewed:** October 2026
