# Complete Raspberry Pi OS Setup to Water Meter Project Deployment

> **Target:** Raspberry Pi 3B+/4/5  
> **OS:** Raspberry Pi OS Trixie 64-bit (Debian 13)  
> **Goal:** Fresh Pi OS → SSH + VNC → Full Water Meter Project Ready  
> **Audience:** Beginners to intermediate — no prior Linux/RPi experience needed

---

## Table of Contents

1. [Download & Flash Raspberry Pi OS](#1-download--flash-raspberry-pi-os)
2. [Initial Boot & Configuration](#2-initial-boot--configuration)
3. [Enable SSH (Headless Access)](#3-enable-ssh-headless-access)
4. [Configure Networking (WiFi + mDNS)](#4-configure-networking-wifi--mdns)
5. [Expand Filesystem](#5-expand-filesystem)
6. [System Update & Upgrade](#6-system-update--upgrade)
7. [Install RealVNC Server](#7-install-realvnc-server)
8. [Configure VNC for Headless/Remote Access](#8-configure-vnc-for-headlessremote-access)
9. [Clone Water Meter Project](#9-clone-water-meter-project)
10. [Create Python Virtual Environment](#10-create-python-virtual-environment)
11. [Install All Project Dependencies](#11-install-all-project-dependencies)
12. [Configure Firebase Credentials](#12-configure-firebase-credentials)
13. [Verify Installation](#13-verify-installation)
14. [Optional: Systemd Service for Auto-start](#14-optional-systemd-service-for-auto-start)

---

## 1. Download & Flash Raspberry Pi OS

### 1.1 Download Raspberry Pi Imager
1. Go to: https://www.raspberrypi.com/software/
2. Download **Raspberry Pi Imager** for your OS (Windows/macOS/Linux)
3. Install and launch

### 1.2 Select OS Image
1. **Choose Device:** Your Pi model (Pi 4/5 → Raspberry Pi 4/5; Pi 3B+ → Raspberry Pi 3)
2. **Choose OS:** 
   - Click **Operating System** → **Raspberry Pi OS (Other)** → **Raspberry Pi OS (64-bit)**
   - ⚠️ **Important:** Select the **64-bit** version (required for XGBoost/ML)
   - Version: **Trixie (Debian 13)** — latest as of 2026
3. **Choose Storage:** Your microSD card (minimum 16 GB, recommended 32 GB+)

### 1.3 Pre-configure OS Settings (Before Flashing)
Click the **gear icon (⚙️ Advanced Options)** or press `Ctrl+Shift+X`:

| Setting | Value | Why |
|---------|-------|-----|
| **Hostname** | `water-meter` | Access via `water-meter.local` |
| **Enable SSH** | ✅ **Checked** | Remote terminal access |
| **SSH Password Auth** | ✅ **Checked** | Use password (not keys) |
| **Username** | `pi` | Default user |
| **Password** | `your-strong-password` | **Change from default!** |
| **Configure Wireless LAN** | ✅ **Checked** | Pre-configure WiFi |
| **SSID** | `YourWiFiName` | Your router's WiFi name |
| **Password** | `YourWiFiPassword` | Your WiFi password |
| **Wireless Country** | `PH` (or your country) | Regulatory domain |
| **Locale** | `en_US.UTF-8` | Language/encoding |
| **Timezone** | `Asia/Manila` (or your timezone) | Correct timestamps |
| **Keyboard Layout** | `us` | Standard US layout |

> 📸 **Screenshot Placeholder:** *Raspberry Pi Imager Advanced Options showing all settings configured*

### 1.4 Flash the SD Card
1. Click **Write**
2. Confirm overwrite warning
3. Wait for "Write Successful" (2-5 minutes)
4. Eject SD card safely

---

## 2. Initial Boot & Configuration

### 2.1 First Boot
1. Insert SD card into Pi
2. Connect power (5V 3A+ USB-C for Pi 4/5; micro-USB for Pi 3B+)
3. Wait 60-90 seconds for first boot (auto-resize, SSH key gen)

### 2.2 Verify SSH Access
From your computer (Linux/macOS/Windows 10+):

```bash
# Test mDNS hostname (preferred)
ssh pi@water-meter.local

# If mDNS fails, find IP via router or:
ssh pi@192.168.1.xxx
```

**First connection:** Type `yes` to accept host key, then enter your password.

> ⚠️ **Windows Users:** If `water-meter.local` doesn't resolve, install [Bonjour Print Services](https://support.apple.com/kb/DL999) or use IP address.

---

## 3. Enable SSH (If Not Done in Imager)

> **Skip if you configured SSH in Raspberry Pi Imager**

```bash
# On Pi (with monitor/keyboard) or via temporary connection:
sudo raspi-config
# Navigate: Interface Options → SSH → Yes → OK → Finish
sudo systemctl enable ssh
sudo systemctl start ssh
```

**Verify:**
```bash
systemctl status ssh
# Should show: Active: active (running)
```

---

## 4. Configure Networking (WiFi + mDNS)

### 4.1 Verify WiFi Connection
```bash
# Check IP address
hostname -I
# Should show: 192.168.1.xxx (your Pi's IP)

# Test internet
ping -c 3 8.8.8.8
ping -c 3 google.com
```

### 4.2 Set Static IP (Optional but Recommended)
**Option A: Router DHCP Reservation (Best)**
1. Log into router admin (192.168.1.1)
2. Find **DHCP Reservation** / **Address Reservation**
3. Add: MAC address of Pi → IP `192.168.1.100` (outside DHCP pool)
4. Reboot Pi

**Option B: nmcli on Pi**
```bash
# Show connections
nmcli con show

# Set static IP for WiFi (replace "YourWiFiSSID" with actual name)
sudo nmcli con mod "YourWiFiSSID" \
    ipv4.addresses 192.168.1.100/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.dns "192.168.1.1, 8.8.8.8" \
    ipv4.method manual

sudo nmcli con up "YourWiFiSSID"
```

### 4.3 Verify mDNS (hostname.local)
```bash
# From your computer:
ping water-meter.local
# Should resolve to Pi's IP

# On Pi - check Avahi:
systemctl status avahi-daemon
# Should show: active (running)
```

---

## 5. Expand Filesystem

> **Usually automatic on first boot**, but verify:

```bash
# Check disk usage
df -h /
# Should show full SD card size (e.g., 29G on 32GB card)

# If not expanded:
sudo raspi-config
# Advanced Options → Expand Filesystem → OK → Reboot
```

---

## 6. System Update & Upgrade

```bash
# Update package lists
sudo apt update

# Full upgrade (includes kernel, firmware)
sudo apt full-upgrade -y

# Install essential tools
sudo apt install -y \
    git \
    curl \
    wget \
    vim \
    htop \
    tree \
    python3-pip \
    python3-venv \
    python3-dev \
    build-essential \
    libopenblas-dev \
    libatlas-base-dev \
    pkg-config \
    cmake

# Clean up
sudo apt autoremove -y
sudo apt clean

# Reboot if kernel updated
sudo reboot
```

**Wait for reboot, then reconnect:**
```bash
ssh pi@water-meter.local
```

---

## 7. Install RealVNC Server

### 7.1 Install via APT (Recommended for Pi OS)
```bash
# RealVNC is pre-installed on Raspberry Pi OS Desktop
# Verify:
dpkg -l | grep realvnc

# If not installed (Lite version):
sudo apt update
sudo apt install -y realvnc-vnc-server realvnc-vnc-viewer
```

### 7.2 Enable VNC Server
```bash
sudo raspi-config
# Interface Options → VNC → Yes → OK → Finish
```

Or via systemctl:
```bash
sudo systemctl enable vncserver-x11-serviced
sudo systemctl start vncserver-x11-serviced

# Check status
systemctl status vncserver-x11-serviced
```

---

## 8. Configure VNC for Headless/Remote Access

### 8.1 Set VNC Password (Required for Headless)
```bash
# Set VNC password (different from user password)
sudo -u pi vncpasswd -service
# Enter password twice (8 chars max)

# Restart VNC service
sudo systemctl restart vncserver-x11-serviced
```

### 8.2 Configure for Headless (No Monitor)
If running **headless** (no HDMI monitor):

```bash
# Enable virtual desktop
sudo raspi-config
# Advanced Options → VNC Resolution → 1920x1080 (or your preferred)
# OR via config.txt:
echo "hdmi_force_hotplug=1" | sudo tee -a /boot/firmware/config.txt
echo "hdmi_group=2" | sudo tee -a /boot/firmware/config.txt
echo "hdmi_mode=82" | sudo tee -a /boot/firmware/config.txt  # 1080p 60Hz

sudo reboot
```

### 8.3 Connect via VNC Viewer
1. Download **VNC Viewer** on your computer: https://www.realvnc.com/en/connect/download/viewer/
2. Open VNC Viewer
3. Enter: `water-meter.local` or `192.168.1.100`
4. Enter VNC password (set in 8.1)

> 📸 **Screenshot Placeholder:** *VNC Viewer connecting to water-meter.local*

---

## 9. Clone Water Meter Project

```bash
# Via SSH or VNC terminal
cd /home/pi

# Clone repository
git clone https://github.com/qppd/wmldad.git
cd wmldad

# Verify structure
ls -la
# Should see: docs/, rpi/, esp32/, etc.
```

---

## 10. Create Python Virtual Environment

```bash
cd /home/pi/wmldad/rpi

# Create virtual environment
python3 -m venv venv

# Activate
source venv/bin/activate

# Verify
which python
# /home/pi/wmldad/rpi/venv/bin/python

python --version
# Python 3.12.x

# Upgrade pip
pip install --upgrade pip setuptools wheel
```

### 10.1 Make Activation Persistent (Optional)
```bash
# Add to .bashrc for auto-activation on SSH login
echo 'source /home/pi/wmldad/rpi/venv/bin/activate' >> ~/.bashrc
source ~/.bashrc
```

---

## 11. Install All Project Dependencies

### 11.1 Create requirements.txt
```bash
cd /home/pi/wmldad/rpi

cat > requirements.txt << 'EOF'
# ML Dependencies
xgboost==2.0.3
scikit-learn==1.3.2
pandas==2.1.4
numpy==1.24.3
joblib==1.3.2

# Web Dependencies
flask==3.0.0
pyrebase4==4.5.0
gunicorn==21.2.0
python-dotenv==1.0.0
requests==2.31.0
EOF
```

### 11.2 Install Dependencies
```bash
# Activate venv if not already
source /home/pi/wmldad/rpi/venv/bin/activate

# Install (takes 5-10 minutes on Pi - compiling xgboost/numpy)
pip install --no-cache-dir -r requirements.txt

# Verify
python -c "
import xgboost, sklearn, pandas, numpy, flask, pyrebase
print('✅ All packages installed successfully')
print(f'XGBoost: {xgboost.__version__}')
print(f'sklearn: {sklearn.__version__}')
print(f'Flask: {flask.__version__}')
"
```

---

## 12. Configure Firebase Credentials

### 12.1 Get Firebase Web Config
1. Open [Firebase Console](https://console.firebase.google.com)
2. Select your project
3. **Project Settings** → **General** → **Your apps** → **Web app (</>)**
4. Copy the `firebaseConfig` object

### 12.2 Create firebase_config.json
```bash
cd /home/pi/wmldad/rpi

cat > firebase_config.json << 'EOF'
{
  "apiKey": "YOUR_API_KEY_HERE",
  "authDomain": "your-project.firebaseapp.com",
  "databaseURL": "https://your-project-default-rtdb.asia-southeast1.firebasedatabase.app",
  "projectId": "your-project",
  "storageBucket": "your-project.appspot.com",
  "messagingSenderId": "123456789",
  "appId": "1:123456789:web:abcdef123456"
}
EOF
```

> **Replace ALL values** with your actual Firebase config.

### 12.3 Create .env File
```bash
cat > .env << 'EOF'
FIREBASE_EMAIL=esp32@your-project.iam.gserviceaccount.com
FIREBASE_PASSWORD=your-strong-password-here
DEVICE_ID=wm_001
FLASK_HOST=0.0.0.0
FLASK_PORT=5000
EOF
```

> **Important:** The email/password must match a user created in Firebase Console → **Authentication** → **Sign-in method** → **Email/Password** → **Add user**.

### 12.4 Verify Config
```bash
python3 -c "
import json, os
from dotenv import load_dotenv
load_dotenv()

with open('firebase_config.json') as f:
    config = json.load(f)
print('Firebase config loaded:')
for k, v in config.items():
    print(f'  {k}: {v[:20]}...' if len(v) > 20 else f'  {k}: {v}')

print()
print('Environment variables:')
print(f'  FIREBASE_EMAIL: {os.getenv(\"FIREBASE_EMAIL\")}')
print(f'  DEVICE_ID: {os.getenv(\"DEVICE_ID\")}')
"
```

---

## 13. Verify Installation

### 13.1 Copy ML Models (If Available)
```bash
# If you have trained models from ml-complete-guide.md:
scp -r models/ pi@water-meter.local:~/wmldad/rpi/models/

# Or create placeholders for testing:
cd /home/pi/wmldad/rpi
python3 << 'PYEOF'
import xgboost as xgb
import numpy as np
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import RobustScaler
import joblib
import json

os.makedirs('models', exist_ok=True)

# Dummy XGBoost
xgb_model = xgb.XGBClassifier()
xgb_model.fit(np.random.rand(100, 9), np.random.randint(0, 3, 100))
xgb_model.save_model('models/xgboost_model.json')

# Dummy Isolation Forest
iforest = IsolationForest(contamination=0.05, random_state=42)
iforest.fit(np.random.rand(100, 9))
joblib.dump(iforest, 'models/isolation_forest.pkl')

# Dummy Scaler
scaler = RobustScaler()
scaler.fit(np.random.rand(100, 9))
joblib.dump(scaler, 'models/scaler.pkl')

# Threshold & features
joblib.dump(-0.5, 'models/iso_threshold.pkl')
feature_cols = ['flow_rate', 'duration', 'hour', 'day', 'fixture_id',
                'inlet_ratio', 'rate_variance', 'is_night', 'pulse_trend']
joblib.dump(feature_cols, 'models/feature_cols.pkl')

with open('models/metadata.json', 'w') as f:
    json.dump({'version': '1.0-placeholder', 'note': 'Replace with real models'}, f)

print('Placeholder models created')
PYEOF
```

### 13.2 Test ML Inference
```bash
source /home/pi/wmldad/rpi/venv/bin/activate
cd /home/pi/wmldad/rpi

python3 -c "
from ml_inference import load_deployment_package
package = load_deployment_package('models')
detector = package['detector']
print('Model loaded:', detector.model_loaded)
print('Features:', detector.n_features)
print('Benchmark:', detector.benchmark(10))
"
```

### 13.3 Test Flask App Manually
```bash
# Terminal 1: Run Flask
source /home/pi/wmldad/rpi/venv/bin/activate
cd /home/pi/wmldad/rpi
python app.py
# Should show: * Running on all addresses (0.0.0.0:5000)
```

**Terminal 2: Test endpoints**
```bash
# Health check
curl http://localhost:5000/api/health
# {"status":"healthy","firebase_connected":true,"model_loaded":true,"device_id":"wm_001"}

# Dashboard
# Open browser: http://water-meter.local:5000/
```

### 13.4 Test Firebase Connection
```bash
# Check ESP32 data arriving
# Firebase Console → Realtime Database → /readings/wm_001/
# Should update every 5 seconds
```

---

## 14. Optional: Systemd Service for Auto-start

### 14.1 Create Service File
```bash
sudo tee /etc/systemd/system/water-meter.service > /dev/null << 'EOF'
[Unit]
Description=Water Meter Leak Detection Backend
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/wmldad/rpi
Environment=PATH=/home/pi/wmldad/rpi/venv/bin
ExecStart=/home/pi/wmldad/rpi/venv/bin/python app.py
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

### 14.2 Enable & Start
```bash
sudo systemctl daemon-reload
sudo systemctl enable water-meter.service
sudo systemctl start water-meter.service

# Check status
sudo systemctl status water-meter.service

# View logs
journalctl -u water-meter.service -f
```

### 14.3 Test Auto-start on Reboot
```bash
sudo reboot
# Wait 60 seconds, then:
curl http://water-meter.local:5000/api/health
# Should return healthy status
```

---

## Quick Reference: Complete Command Summary

```bash
# ===== ONE-LINE SETUP (after SSH access) =====
# Run each section separately, not as one script

# 1. Update & tools
sudo apt update && sudo apt full-upgrade -y && sudo apt install -y git python3-venv python3-dev build-essential libopenblas-dev libatlas-base-dev

# 2. Clone project
cd /home/pi && git clone https://github.com/qppd/wmldad.git && cd wmldad/rpi

# 3. Virtual environment
python3 -m venv venv && source venv/bin/activate

# 4. Dependencies
cat > requirements.txt << 'EOF'
xgboost==2.0.3
scikit-learn==1.3.2
pandas==2.1.4
numpy==1.24.3
joblib==1.3.2
flask==3.0.0
pyrebase4==4.5.0
gunicorn==21.2.0
python-dotenv==1.0.0
requests==2.31.0
EOF
pip install --no-cache-dir -r requirements.txt

# 5. Firebase config
cat > firebase_config.json << 'EOF'
{ "apiKey": "YOUR_KEY", "authDomain": "proj.firebaseapp.com", "databaseURL": "https://proj-default-rtdb.region.firebasedatabase.app", "projectId": "proj", "storageBucket": "proj.appspot.com", "messagingSenderId": "123", "appId": "1:123:web:abc" }
EOF

cat > .env << 'EOF'
FIREBASE_EMAIL=esp32@proj.iam.gserviceaccount.com
FIREBASE_PASSWORD=your-password
DEVICE_ID=wm_001
FLASK_HOST=0.0.0.0
FLASK_PORT=5000
EOF

# 6. Models (placeholder or real)
mkdir -p models
python3 -c "
import xgboost as xgb, numpy as np
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import RobustScaler
import joblib, json, os
os.makedirs('models', exist_ok=True)
xgb.XGBClassifier().fit(np.random.rand(100,9), np.random.randint(0,3,100)).save_model('models/xgboost_model.json')
joblib.dump(IsolationForest(contamination=0.05,random_state=42).fit(np.random.rand(100,9)), 'models/isolation_forest.pkl')
joblib.dump(RobustScaler().fit(np.random.rand(100,9)), 'models/scaler.pkl')
joblib.dump(-0.5, 'models/iso_threshold.pkl')
joblib.dump(['flow_rate','duration','hour','day','fixture_id','inlet_ratio','rate_variance','is_night','pulse_trend'], 'models/feature_cols.pkl')
json.dump({'version':'placeholder'}, open('models/metadata.json','w'))
"

# 7. Test
python -c "from ml_inference import load_deployment_package; d=load_deployment_package('models'); print('OK:', d['detector'].benchmark(5))"

# 8. Run (manual)
python app.py
# Visit http://water-meter.local:5000/

# 9. Auto-start (optional)
sudo tee /etc/systemd/system/water-meter.service > /dev/null << 'EOF'
[Unit]
Description=Water Meter Backend
After=network.target
[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/wmldad/rpi
Environment=PATH=/home/pi/wmldad/rpi/venv/bin
ExecStart=/home/pi/wmldad/rpi/venv/bin/python app.py
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable water-meter && sudo systemctl start water-meter
```

---

## Troubleshooting Quick Reference

| Issue | Solution |
|-------|----------|
| **Can't SSH** | Check IP: `hostname -I` on Pi. Use IP not hostname. Check `systemctl status ssh`. |
| **VNC "Cannot connect"** | Ensure VNC enabled: `sudo raspi-config` → Interface Options → VNC. Set password: `sudo -u pi vncpasswd -service`. |
| **pip install fails (xgboost)** | `sudo apt install -y libopenblas-dev libatlas-base-dev gfortran` then retry. |
| **Firebase permission denied** | Check Security Rules in Firebase Console. Verify email/password user exists in Auth. |
| **Module not found** | Ensure venv activated: `source /home/pi/wmldad/rpi/venv/bin/activate`. |
| **Port 5000 in use** | `sudo lsof -i :5000` → kill process, or change `FLASK_PORT` in `.env`. |
| **SD card full** | `df -h` → clean with `sudo apt clean`, `sudo journalctl --vacuum-time=7d`. |
| **VNC headless no display** | Add to `/boot/firmware/config.txt`: `hdmi_force_hotplug=1`, `hdmi_group=2`, `hdmi_mode=82`. Reboot. |

---

## Next Steps After Setup

1. **Hardware:** Wire ESP32 + 4× YF-S201 per [block-diagram.md](./docs/block-diagram.md)
2. **Firmware:** Flash ESP32 per [esp32-setup-guide.md](./docs/esp32-setup-guide.md)
3. **Calibrate:** Bucket test per [calibration.md](./docs/calibration.md)
4. **ML Training:** Follow [ml-complete-guide.md](./docs/ml-complete-guide.md) for real models
5. **Remote Access:** Configure Cloudflare Tunnel for HTTPS access from anywhere

---

*Last updated: July 2026 | Target: Raspberry Pi OS Trixie 64-bit (Debian 13) | Compatible with Pi 3B+/4/5*