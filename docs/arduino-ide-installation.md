# Arduino IDE Installation on Raspberry Pi OS (64-bit)

> **Target:** Raspberry Pi OS Bookworm/Trixie (64-bit) on Pi 3B+/4/5  
> **Purpose:** GUI-based ESP32 firmware development and upload  
> **Method:** `pip install arduino` (official Arduino CLI + IDE package)

---

## Install Arduino IDE via pip

```bash
# Install Arduino IDE (includes Arduino CLI + IDE 2.x)
pip install arduino

# Verify installation
arduino --version
# Arduino IDE 2.3.x
```

That's it. The `arduino` package from PyPI installs the official Arduino IDE 2.x with CLI included.

---

## ESP32 Board Support Setup

### 1. Open Preferences
1. Launch Arduino IDE: `arduino` (or from Applications menu)
2. **File** → **Preferences** (`Ctrl+,`)
3. **Additional Boards Manager URLs:** paste:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
4. Click **OK**

### 2. Install ESP32 Core
1. **Tools** → **Board** → **Boards Manager...** (`Ctrl+Shift+B`)
2. Search: **esp32**
3. Click **Install** on **"esp32 by Espressif Systems"**
4. Wait for download (~200 MB toolchain)

### 3. Select Your Board
**Tools** → **Board** → **ESP32 Arduino** → **NodeMCU-32S**

---

## Library Installation

Install required libraries via Library Manager (**Tools** → **Manage Libraries...** / `Ctrl+Shift+I`):

| Library | Version |
|---------|---------|
| **Firebase ESP Client** (by mobizt) | ≥ 4.4.x |

> **Note:** `Firebase-ESP-Client` by Mobizt already bundles/handles JSON serialization internally. No separate `ArduinoJson` library installation needed.

---

## Serial Port Permissions

```bash
# Add user to dialout group for USB serial access
sudo usermod -a -G dialout $USER
newgrp dialout  # Apply immediately (or log out/in)

# Verify
groups $USER  # Should include: dialout
```

### Optional: udev Rule for Consistent Device Name

```bash
sudo tee /etc/udev/rules.d/99-esp32.rules > /dev/null <<'EOF'
# CP2102/CP2104 (NodeMCU-32S)
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", MODE="0666", GROUP="dialout", SYMLINK+="ttyESP32"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
```

Now ESP32 appears as `/dev/ttyESP32` consistently.

---

## Quick Test

### 1. Blink Sketch
```cpp
// File → Examples → 01.Basics → Blink
void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
  Serial.begin(115200);
  while (!Serial) delay(10);
  Serial.println("ESP32 Blink Test");
}
void loop() {
  digitalWrite(LED_BUILTIN, HIGH); Serial.println("ON"); delay(500);
  digitalWrite(LED_BUILTIN, LOW);  Serial.println("OFF"); delay(500);
}
```

### 2. Connect ESP32
- Plug ESP32 via USB **data cable** (not charge-only!)
- **Tools** → **Port** → Select `/dev/ttyUSB0` or `/dev/ttyESP32`

### 3. Upload
- Click **Upload** (`Ctrl+U`) or Sketch → Upload
- Wait for "Done uploading"

### 4. Serial Monitor
- **Tools** → **Serial Monitor** (`Ctrl+Shift+M`)
- Set baud: **115200**
- Should see "ON"/"OFF" every 500ms

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Board not found" / No device on `/dev/ttyUSB0` | Use **data cable** (not charge-only). Check `ls /dev/tty*`. Hold **BOOT** → press **EN** → release **BOOT** → upload. |
| "Error compiling for board NodeMCU-32S" | Boards Manager → esp32 → Remove → Install |
| Slow IDE startup / High memory | `sudo fallocate -l 2G /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |
| IDE can't access serial ports | `flatpak permission-set device serial cc.arduino.IDE2 yes` (if using Flatpak) — but `pip install arduino` doesn't need this |
| Library not found after install | Restart Arduino IDE. Check **Sketch** → **Include Library** |
| "esptool.py not found" during upload | `pip3 install esptool` |

---

## Quick Reference

| Task | Command / Menu |
|------|----------------|
| Launch IDE | `arduino` |
| Preferences | File → Preferences (`Ctrl+,`) |
| Boards Manager | Tools → Board → Boards Manager (`Ctrl+Shift+B`) |
| Library Manager | Tools → Manage Libraries (`Ctrl+Shift+I`) |
| Select Board | Tools → Board → ESP32 Arduino → NodeMCU-32S |
| Select Port | Tools → Port → `/dev/ttyUSB0` or `/dev/ttyESP32` |
| Verify/Compile | Sketch → Verify/Compile (`Ctrl+R`) |
| Upload | Sketch → Upload (`Ctrl+U`) |
| Serial Monitor | Tools → Serial Monitor (`Ctrl+Shift+M`) |
| Board Manager URL | Preferences → Additional Boards Manager URLs |

---

## Official References

- [Arduino IDE Download](https://www.arduino.cc/en/software)
- [ESP32 Arduino Core Installation](https://docs.espressif.com/projects/arduino-esp32/en/latest/installing.html)
- [Arduino PyPI Package](https://pypi.org/project/arduino/)
- [Raspberry Pi Arduino Forum](https://forums.raspberrypi.com/viewforum.php?f=107)

---

*Last updated: July 2026 | `pip install arduino` on Raspberry Pi OS Bookworm/Trixie (64-bit) | Compatible with Pi 3B+/4/5*