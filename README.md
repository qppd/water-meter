# WMLDAD — Smart Water Monitoring System

> **A Research Project** — Smart Water Monitoring System that detects leaks, anomalies, and per-fixture consumption using ESP32, Firebase, Raspberry Pi, and Machine Learning (XGBoost).

---

## Developer Quick-Start: Step-by-Step Process

Follow these steps **in order**. Each step links to the detailed guide.

### Phase 1: Prepare (Do First)

| Step | Action | Guide | Est. Time |
|------|--------|-------|-----------|
| 1 | **Buy parts** — Order from BOM (Makerlab Electronics on Shopee/Lazada) | [BOM.md](./docs/bom.md) | 1–2 weeks shipping |
| 2 | **Flash Raspberry Pi OS** — Trixie 64-bit, enable SSH + WiFi in Imager | [raspberry-pi-installation.md](./docs/raspberry-pi-installation.md) | 30 min |
| 3 | **Create Firebase project** — Realtime DB + Email/Password auth + web app config | [firebase-setup-guide.md](./docs/firebase-setup-guide.md) | 20 min |

> ⚠️ **Do Step 1–3 in parallel.** Hardware shipping takes longest.

---

### Phase 2: Hardware Assembly

| Step | Action | Guide | Est. Time |
|------|--------|-------|-----------|
| 4 | **Wire ESP32 + 4× YF-S201** on expansion board (GPIO 26, 25, 33, 32) | [block-diagram.md](./docs/block-diagram.md#pin-connections) | 1 hr |
| 5 | **Plumbing** — Install sensors in-line with check valves (arrow = flow direction) | [setup.md#phase-4-hardware-assembly](./docs/setup.md#phase-4-hardware-assembly) | 2–4 hrs |
| 6 | **Enclosure** — Mount in IP67 box with cable glands | [block-diagram.md](./docs/block-diagram.md#component-layout-enclosure) | 1 hr |

---

### Phase 3: ESP32 Firmware

| Step | Action | Guide | Est. Time |
|------|--------|-------|-----------|
| 7 | **Install Arduino IDE 2.x** (Flatpak on RPi, or Windows/macOS) | [arduino-ide-installation.md](./docs/arduino-ide-installation.md) | 15 min |
| 8 | **Add ESP32 board support** — Board Manager URL + install `esp32:esp32` | [esp32-setup-guide.md](./docs/esp32-setup-guide.md#arduino-ide-board-configuration) | 10 min |
| 9 | **Install libraries** — `Firebase ESP Client` (mobizt) + `ArduinoJson` | [firmware.md](./docs/firmware.md#required-libraries) | 5 min |
| 10 | **Configure `config.h`** — WiFi, Firebase API key, DB URL, user email/password, device ID | [setup.md#phase-5-esp32-firmware-upload](./docs/setup.md#phase-5-esp32-firmware-upload) | 10 min |
| 11 | **Upload firmware** — Select NodeMCU-32S, correct port, Upload (Ctrl+U) | [esp32-setup-guide.md](./docs/esp32-setup-guide.md#upload-process--boot-modes) | 5 min |
| 12 | **Verify** — Serial Monitor (115200): WiFi connect → Firebase stream start → sensor ISRs attached | [setup.md](./docs/setup.md#step-53-monitor-serial-output) | 5 min |

---

### Phase 4: Sensor Calibration (Required Before ML)

| Step | Action | Guide | Est. Time |
|------|--------|-------|-----------|
| 13 | **Bucket test each sensor** — 5L measured, calculate PPL, update `config.h` | [calibration.md](./docs/calibration.md) | 30 min/sensor |

> 🎯 Target: < 3% error per sensor. Uncalibrated sensors = false leaks / missed leaks.

---

### Phase 5: Raspberry Pi Backend

| Step | Action | Guide | Est. Time |
|------|--------|-------|-----------|
| 14 | **SSH into RPi** — `ssh pi@water-meter.local` | [raspberry-pi-networking.md](./docs/raspberry-pi-networking.md) | 5 min |
| 15 | **Clone repo + create venv + install deps** | [rpi-backend.md](./docs/rpi-backend.md#quick-start) | 10 min |
| 16 | **Copy `firebase_config.json` + `.env`** (from Firebase web app config) | [rpi-backend.md](./docs/rpi-backend.md#firebase-configuration-pyrebase4) | 5 min |
| 17 | **Run Flask** — `python app.py`, verify dashboard at `http://<rpi-ip>:5000/` | [rpi-backend.md](./docs/rpi-backend.md#running-the-backend) | 5 min |
| 18 | **Enable systemd service** for auto-start on boot | [rpi-backend.md](./docs/rpi-backend.md#production-mode-systemd-service) | 5 min |

---

### Phase 6: ML Model (Can Defer)

| Step | Action | Guide | Est. Time |
|------|--------|-------|-----------|
| 19 | **Train XGBoost + Isolation Forest** — Use Google Colab (GPU) | [ml-training-guide.md](./docs/ml-training-guide.md) | 30 min |
| 20 | **Export models** — `xgboost_model.json`, `isolation_forest.pkl`, `scaler.pkl` to `rpi/models/` | [model-deployment-guide.md](./docs/model-deployment-guide.md) | 5 min |
| 21 | **Verify inference** — Dashboard shows leak classifications | [ml-model.md](./docs/ml-model.md) | 5 min |

> 💡 **Skip for initial bring-up.** System works with local ESP32 rules (inlet balance, continuous flow, drip detection) without ML. Add ML after hardware + data pipeline verified.

---

### Phase 7: Remote Access (Optional)

| Step | Action | Guide | Est. Time |
|------|--------|-------|-----------|
| 22 | **Router port forward** — External 8443 → RPi:5000 | [rpi-backend.md](./docs/rpi-backend.md#remote-access-port-forwarding--ddns) | 10 min |
| 23 | **DDNS** (DuckDNS free) or **Cloudflare Tunnel** (HTTPS) | [rpi-backend.md](./docs/rpi-backend.md#dynamic-dns-optional-but-recommended) | 15 min |

---

## Essential Guides Only (Bookmark These)

| Guide | Purpose |
|-------|---------|
| [BOM.md](./docs/bom.md) | Parts list with Shopee links, prices |
| [raspberry-pi-installation.md](./docs/raspberry-pi-installation.md) | Pi OS Trixie 64-bit + SSH + WiFi |
| [firebase-setup-guide.md](./docs/firebase-setup-guide.md) | Firebase project, auth, web config, security rules |
| [block-diagram.md](./docs/block-diagram.md) | Pinout, wiring, enclosure layout |
| [setup.md](./docs/setup.md) | Full phased walkthrough (reference) |
| [calibration.md](./docs/calibration.md) | Bucket test procedure |
| [arduino-ide-installation.md](./docs/arduino-ide-installation.md) | Flatpak on RPi, board manager, libraries |
| [esp32-setup-guide.md](./docs/esp32-setup-guide.md) | Drivers, board select, boot modes, upload errors |
| [rpi-backend.md](./docs/rpi-backend.md) | Flask + Pyrebase4 + systemd + remote access |
| [firmware.md](./docs/firmware.md) | ESP32 code structure, Firebase-ESP-Client usage |
| [troubleshooting.md](./docs/troubleshooting.md) | Serial commands, LED codes, common fixes |

---

## Guides Removed (Cause Delays)

| Removed Guide | Why |
|---------------|-----|
| `ml-model.md` | Theory — read only if tuning model |
| `ml-dataset-guide.md` | Data engineering — not needed for deployment |
| `ml-training-guide.md` | Colab steps — only needed once for training |
| `model-deployment-guide.md` | Edge optimization — default models work |
| `project-timeline.md` | Student schedule — not for developers |
| `stacks.md` | Version table — reference only |
| `flowchart.md` / `system-architecture.md` | Diagrams — view once for understanding |
| `raspberry-pi-networking.md` | SSH/mDNS — covered in Pi install guide |
| `remote-desktop-guide.md` | RealVNC — not needed for headless operation |

---

## Hardware Summary

| Component | Qty | Key Spec |
|-----------|-----|----------|
| ESP32 38-pin (NodeMCU-32S) | 1 | CP2102 USB-UART |
| ESP32 Expansion Board | 1 | Screw terminals |
| YF-S201 Flow Sensor | 4 | 1/2" NPT, Hall effect |
| Check Valve 1/2" | 3 | Brass/PVC, prevent backflow |
| 12V 5A PSU + LM2596S buck | 1 | 220V → 12V → 5V |
| IP67 ABS Enclosure | 1 | 175×125×75mm |
| JST-XH 3-pin M/F | 4 each | Pre-crimped, no crimp tool |
| Perf board 20×80mm | 2 | Soldered connections |

---

## Quick Verification Checklist

After each phase, verify:

- [ ] **Phase 1:** Pi boots, `ssh pi@water-meter.local` works, Firebase console shows project
- [ ] **Phase 2:** All 4 sensors show pulses in Serial Monitor when water flows
- [ ] **Phase 3:** Firebase `/readings/wm_001/` updates every 5 sec
- [ ] **Phase 4:** 5L bucket test → < 3% error on each sensor
- [ ] **Phase 5:** Dashboard at `:5000` shows live flow rates per fixture
- [ ] **Phase 6:** Leak simulation (slow drip) → alert appears in dashboard
- [ ] **Phase 7:** Remote URL (DDNS:8443 or Cloudflare) loads dashboard

---

## License

MIT

## Author

[qppd](https://github.com/qppd) — Quezon Province, Philippines