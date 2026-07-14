# ESP32 Complete Setup Guide — Raspberry Pi OS Trixie 64-bit (Debian 13)

> **Target:** ESP32 NodeMCU-32S (38-pin) with Expansion Board  
> **Host OS:** Raspberry Pi OS Trixie 64-bit (Debian 13) on Pi 3B+/4/5  
> **Method:** `pip install arduino` (official Arduino CLI + IDE 2.x package)  
> **Audience:** Complete hardware/software setup from unboxing to first upload

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Hardware Overview](#hardware-overview)
3. [Install Arduino IDE on Raspberry Pi OS Trixie](#install-arduino-ide-on-raspberry-pi-os-trixie)
4. [Verify USB Driver (Built into Kernel)](#verify-usb-driver-built-into-kernel)
5. [Configure ESP32 Board Support](#configure-esp32-board-support)
6. [Select Correct Board: NodeMCU-32S](#select-correct-board-nodemcu-32s)
7. [Serial Port Permissions & udev Rules](#serial-port-permissions--udev-rules)
8. [Install Required Libraries](#install-required-libraries)
9. [Upload Process & Boot Modes](#upload-process--boot-modes)
10. [Boot Button & EN Button Usage](#boot-button--en-button-usage)
11. [Common Upload Errors & Fixes](#common-upload-errors--fixes)
12. [Verification Checklist](#verification-checklist)
13. [Quick Reference Card](#quick-reference-card)

---

## Prerequisites

- Raspberry Pi 3B+ / 4 / 5 with **Raspberry Pi OS Trixie 64-bit** (Debian 13)
- Internet connection (for downloading ESP32 toolchain ~200 MB)
- ESP32 NodeMCU-32S + Expansion Board
- **Data-capable micro-USB cable** (not charge-only!)
- Basic Linux terminal familiarity

> 📸 **Screenshot Placeholder:** *Raspberry Pi OS Trixie desktop showing terminal and file manager*

---

## Hardware Overview

### ESP32 NodeMCU-32S (38-pin)

| Feature | Specification |
|---------|---------------|
| **MCU** | ESP32-WROOM-32D (Xtensa LX6 dual-core @ 240 MHz) |
| **USB-UART** | CP2102 (SiLabs) — requires CP210x driver |
| **GPIO** | 38 pins (34 usable) |
| **Flash** | 4 MB |
| **Voltage** | 3.3V logic, 5V USB input |
| **Pinout** | See [Block Diagram](../docs/block-diagram.md#pinout-reference-esp32-38-pin) |

### Expansion Board (38-pin)

- Screw terminals for all GPIOs
- 5V and 3.3V power rails
- Reset (EN) and Boot buttons accessible
- Mounting holes for enclosure

> 📸 **Screenshot Placeholder:** *Photo of ESP32 NodeMCU-32S mounted on expansion board with labeled pins*

---

## Install Arduino IDE on Raspberry Pi OS Trixie

### Method: `pip install arduino` (Recommended)

The official Arduino package on PyPI installs Arduino IDE 2.x with Arduino CLI included.

```bash
# Update system first
sudo apt update && sudo apt full-upgrade -y

# Install pip if not present
sudo apt install -y python3-pip

# Install Arduino IDE (includes Arduino CLI)
pip install arduino

# Verify installation
arduino --version
# Arduino IDE 2.3.x
```

### Launch Arduino IDE

```bash
# From terminal
arduino

# Or from Applications Menu → Programming → Arduino IDE 2
```

> **Why pip?** The `arduino` PyPI package is the official distribution method for Linux ARM64. It provides automatic updates via `pip install --upgrade arduino` and works natively on Raspberry Pi OS Trixie 64-bit without Flatpak sandbox issues.

### Alternative: Flatpak (if pip method fails)

```bash
flatpak install flathub cc.arduino.IDE2
flatpak run cc.arduino.IDE2
# Note: Flatpak requires extra permission for serial ports:
# flatpak permission-set device serial cc.arduino.IDE2 yes
```

---

## Verify USB Driver (Built into Kernel)

On Linux (including Raspberry Pi OS Trixie), the CP210x driver is **built into the kernel** (version 3.x+). No installation needed.

```bash
# Plug in ESP32 via micro-USB data cable
# Check kernel messages
dmesg -w
# Watch for:
# cp210x 1-1.2:1.0: cp210x converter detected
# usb 1-1.2: cp210x converter now attached to ttyUSB0

# Verify device appears
ls /dev/ttyUSB*
# Should show: /dev/ttyUSB0  (or ttyUSB1 if multiple devices)
```

| OS | Driver Status | Verification |
|----|---------------|--------------|
| **Raspberry Pi OS Trixie** | Built-in (kernel) | `ls /dev/ttyUSB*` |
| Ubuntu 22.04+/Debian 12+ | Built-in | `ls /dev/ttyUSB*` |
| Windows | Manual install needed | Device Manager → CP210x |
| macOS | Manual install needed | `ls /dev/tty.*` |

> 📸 **Screenshot Placeholder:** *Terminal showing `dmesg` output with cp210x converter attached to ttyUSB0*

---

## Configure ESP32 Board Support

### 1. Open Preferences

1. Launch Arduino IDE: `arduino`
2. **File** → **Preferences** (`Ctrl+,`)

### 2. Add ESP32 Board Manager URL

In **Additional Boards Manager URLs**, paste:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Click **OK**.

> 📸 **Screenshot Placeholder:** *Arduino IDE Preferences dialog with ESP32 URL pasted in Additional Boards Manager URLs field*

### 3. Install ESP32 Core

1. **Tools** → **Board** → **Boards Manager...** (`Ctrl+Shift+B`)
2. Search: **esp32**
3. Click **Install** on **"esp32 by Espressif Systems"** (latest version)
4. Wait for download (~200 MB toolchain for ARM64)

> 📸 **Screenshot Placeholder:** *Boards Manager showing "esp32 by Espressif Systems" installing with progress bar*

---

## Select Correct Board: NodeMCU-32S

> ⚠️ **Critical:** Selecting the wrong board = wrong pin mapping, wrong flash size, upload failures.

| Selection | Value |
|-----------|-------|
| **Board** | Tools → Board → ESP32 Arduino → **NodeMCU-32S** |
| **FQBN** | `esp32:esp32:nodemcu-32s` |
| **Upload Speed** | 921600 (Tools → Upload Speed) |
| **CPU Frequency** | 240 MHz (WiFi/BT) (Tools → CPU Frequency) |
| **Flash Mode** | QIO (Tools → Flash Mode) |
| **Partition Scheme** | Default 4MB with spiffs (1.2MB APP/1.5MB SPIFFS) |

### Board Variants Reference

| Board | FQBN | Use Case |
|-------|------|----------|
| **NodeMCU-32S** | `esp32:esp32:nodemcu-32s` | **This project** — 38-pin, CP2102 |
| ESP32 Dev Module | `esp32:esp32:esp32dev` | Generic 30-pin dev board |
| ESP32-S3 DevKitC-1 | `esp32:esp32:esp32s3devkitc-1` | ESP32-S3 (different chip) |
| TTGO T-Display | `esp32:esp32:ttgo-tdisplay` | With built-in screen |
| M5Stack Core2 | `esp32:esp32:m5stack-core2` | M5Stack device |

> 📸 **Screenshot Placeholder:** *Tools → Board menu with ESP32 Arduino expanded and NodeMCU-32S highlighted*

---

## Serial Port Permissions & udev Rules

### 1. Add User to dialout Group

```bash
# Required for USB serial access
sudo usermod -a -G dialout $USER
newgrp dialout  # Apply immediately (or log out/in)

# Verify
groups $USER
# Should include: dialout
```

### 2. Create udev Rule for Persistent Device Name

```bash
# Create rule for CP2102 (NodeMCU-32S)
sudo tee /etc/udev/rules.d/99-esp32.rules > /dev/null <<'EOF'
# CP2102/CP2104 (NodeMCU-32S)
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", MODE="0666", GROUP="dialout", SYMLINK+="ttyESP32"
# CH340 (some ESP32 boards)
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", MODE="0666", GROUP="dialout", SYMLINK+="ttyESP32"
EOF

# Reload rules
sudo udevadm control --reload-rules
sudo udevadm trigger

# Verify symlink created
ls -l /dev/ttyESP32
# /dev/ttyESP32 -> /dev/ttyUSB0
```

Now the ESP32 is **always** accessible at `/dev/ttyESP32` regardless of whether it's ttyUSB0 or ttyUSB1.

---

## Install Required Libraries

Open **Library Manager**: **Tools** → **Manage Libraries...** (`Ctrl+Shift+I`)

| Library | Version | Purpose |
|---------|---------|---------|
| **Firebase ESP Client** (by mobizt) | ≥ 4.4.x | Firebase Realtime DB (push, set, stream) |

> **Note:** `Firebase-ESP-Client` by Mobizt already bundles/handles JSON serialization internally. No separate `ArduinoJson` library installation needed.

> 📸 **Screenshot Placeholder:** *Library Manager showing "Firebase ESP Client" by mobizt installing*

---

## Upload Process & Boot Modes

### Normal Upload (Automatic)

1. Connect ESP32 via micro-USB **data cable**
2. Select **Board**: NodeMCU-32S
3. Select **Port**: `/dev/ttyESP32` (or `/dev/ttyUSB0`)
4. Click **Upload** (`Ctrl+U`)
5. Arduino IDE automatically:
   - Resets ESP32 into bootloader mode (via DTR/RTS)
   - Uploads firmware via esptool.py
   - Resets into application mode

### Manual Bootloader Mode (If Auto Fails)

**When to use:** Upload fails with "Failed to connect to ESP32" or "Timed out waiting for packet header"

#### BOOT + EN Button Sequence

```
1. Hold BOOT button (GPIO 0 low)
2. Press and release EN (Reset) button
3. Release BOOT button
4. ESP32 is now in bootloader mode (waiting for upload)
5. Click Upload immediately
```

> 📸 **Screenshot Placeholder:** *Photo of ESP32 expansion board with BOOT and EN buttons labeled*

### Boot Mode Pin States

| Mode | GPIO 0 (BOOT) | GPIO 2 | EN (Reset) | Use Case |
|------|---------------|--------|------------|----------|
| **Normal Boot** | High (1) | Don't care | Running | Application runs |
| **Bootloader** | Low (0) | Don't care | Pulse low | Firmware upload |
| **Download Boot** | Low (0) | Low (0) | Pulse low | Factory test (avoid) |

---

## Boot Button & EN Button Usage

### Buttons on Expansion Board

| Button | GPIO | Function |
|--------|------|----------|
| **BOOT** | GPIO 0 | Hold during reset → bootloader mode |
| **EN** (Reset) | EN (CHIP_PU) | Hardware reset |

### Common Scenarios

| Scenario | Action |
|----------|--------|
| **Normal upload** | Just click Upload (auto-reset works) |
| **Upload fails** | Hold BOOT → Press EN → Release BOOT → Upload |
| **Stuck in bootloader** | Press EN alone (reset) |
| **Need clean boot** | Press EN alone |
| **Factory reset** | Hold BOOT + EN for 5s → Release both |

---

## Common Upload Errors & Fixes

### Error 1: "Failed to connect to ESP32: Timed out waiting for packet header"

| Cause | Fix |
|-------|-----|
| Not in bootloader mode | Hold BOOT → Press EN → Release BOOT → Upload |
| Wrong port selected | Verify port: `ls /dev/ttyUSB*` or use `/dev/ttyESP32` |
| Charge-only USB cable | Use **data cable** (test: phone file transfer works) |
| Driver not loaded | `lsmod \| grep cp210x` — should show cp210x module |
| Port busy (Serial Monitor open) | Close Serial Monitor / Plotter before upload |

### Error 2: "Permission denied" on `/dev/ttyUSB0`

```bash
sudo usermod -a -G dialout $USER
newgrp dialout
# Or logout/login
```

### Error 3: "esptool.py not found" / "python3: not found"

```bash
pip3 install esptool
which python3
```

### Error 4: Wrong upload speed / baud rate

- Arduino IDE: **Tools** → **Upload Speed** → **921600** (or 115200 for reliability)
- Serial Monitor: **115200** (must match `Serial.begin(115200)` in code)

### Error 5: "Flash size mismatch" / "Invalid head of packet"

| Cause | Fix |
|-------|-----|
| Wrong board selected | Select **NodeMCU-32S** (not ESP32 Dev Module) |
| Corrupt flash | `esptool.py --port /dev/ttyESP32 erase_flash` then re-upload |
| Partition scheme mismatch | Tools → Partition Scheme → Default 4MB with spiffs |

### Error 6: "Brownout detector triggered" (Random resets)

| Cause | Fix |
|-------|-----|
| Insufficient power | Use 5V 2A+ supply; add 1000µF capacitor on 5V rail |
| Thin USB cable | Use thick, short, quality USB cable |
| Powered via laptop USB | Use powered hub or wall adapter |

### Error 7: "MD5 of file does not match data in flash!"

```bash
esptool.py --port /dev/ttyESP32 erase_flash
# Then upload again via Arduino IDE
```

### Error 8: Upload succeeds but Serial Monitor shows garbage

| Cause | Fix |
|-------|-----|
| Wrong baud rate | Set Serial Monitor to **115200** |
| Line ending | Set to "Both NL & CR" or "Newline" |
| Corrupt firmware | Erase flash and re-upload |

---

## Verification Checklist

After successful upload, verify:

### 1. Serial Output (115200 baud)

```
ets Jun  8 2016 00:22:57
rst:0x1 (POWERON_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00,hd_drv:0x00,wp_drv:0x00
mode:DIO, clock div:2
load:0x3fff0030,len:1184
load:0x40078000,len:13424
load:0x40080400,len:3600
entry 0x400805e0
Connecting to WiFi...
WiFi connected! IP: 192.168.1.100
Firebase initialized successfully
Starting stream on: /commands/wm_001
Sensor 0 (inlet): ISR attached on GPIO 26
Sensor 1 (fix1): ISR attached on GPIO 25
Sensor 2 (fix2): ISR attached on GPIO 33
Sensor 3 (fix3): ISR attached on GPIO 32
Reading: inlet=0.00 L/min fix1=0.00 L/min fix2=0.00 L/min fix3=0.00 L/min
Data uploaded to Firebase
```

### 2. Firebase Console

1. Open Firebase Console → Realtime Database
2. Check `/readings/wm_001/` — should see timestamped data every 5s

### 3. LED Indicators (Built-in LED on GPIO 2)

| Pattern | Meaning |
|---------|---------|
| Solid green | Normal operation |
| Blink green (1s) | WiFi connecting |
| Blink blue (fast) | Transmitting to Firebase |
| Solid yellow | Minor leak detected |
| Solid red | Major leak detected |
| Red flash | Upload failed |
| White blink (3x) | Upload success |
| Off | Deep sleep / no power |

### 4. Sensor Test

1. Blow into inlet sensor → Should show flow rate > 0
2. Tap each fixture sensor → Individual readings change
3. Check `inlet ≈ fix1 + fix2 + fix3` (within 10%)

---

## Quick Reference Card

| Task | Command / Menu |
|------|----------------|
| **Launch IDE** | `arduino` |
| **Preferences** | File → Preferences (`Ctrl+,`) |
| **Boards Manager** | Tools → Board → Boards Manager (`Ctrl+Shift+B`) |
| **Library Manager** | Tools → Manage Libraries (`Ctrl+Shift+I`) |
| **Select Board** | Tools → Board → ESP32 Arduino → **NodeMCU-32S** |
| **Select Port** | Tools → Port → `/dev/ttyESP32` |
| **Upload Speed** | Tools → Upload Speed → 921600 |
| **Verify/Compile** | Sketch → Verify/Compile (`Ctrl+R`) |
| **Upload** | Sketch → Upload (`Ctrl+U`) |
| **Serial Monitor** | Tools → Serial Monitor (`Ctrl+Shift+M`) |
| **Board Manager URL** | Preferences → Additional Boards Manager URLs |
| **Bootloader Mode** | Hold BOOT → Press EN → Release BOOT |
| **Erase Flash** | `esptool.py --port /dev/ttyESP32 erase_flash` |
| **Add to dialout** | `sudo usermod -a -G dialout $USER && newgrp dialout` |

---

## Official References

- [ESP32 Arduino Core Installation](https://docs.espressif.com/projects/arduino-esp32/en/latest/installing.html)
- [Espressif Board Manager JSON](https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json)
- [ESP32 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf)
- [NodeMCU-32S Pinout](https://github.com/nodemcu/nodemcu-devkit-v1.0)
- [CP210x Drivers](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers)
- [esptool.py Documentation](https://docs.espressif.com/projects/esptool/en/latest/esp32/)
- [Arduino Forum - ESP32](https://forum.arduino.cc/c/hardware/esp32/61)
- [Arduino PyPI Package](https://pypi.org/project/arduino/)

---

## Next Steps

Proceed to:
1. [Firebase ESP Client Guide](./firebase-esp-client-guide.md) — Library setup and usage
2. [Project Setup Guide](./setup.md) — Full system deployment
3. [Calibration Guide](./calibration.md) — Sensor K-factor calibration
4. [ESP32 ↔ RPi Communication](./esp32-rpi-communication.md) — USB serial data flow

---

*Last updated: July 2026 | Tested with ESP32 NodeMCU-32S, Arduino IDE 2.3.x (via `pip install arduino`), ESP32 Core 3.x, Raspberry Pi OS Trixie 64-bit (Debian 13) | Compatible with Pi 3B+/4/5*