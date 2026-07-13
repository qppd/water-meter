# RPi Backend App — Deploying the Flask + ML Backend on Raspberry Pi

> **Hardware:** Raspberry Pi 3B+/4/5  
> **OS:** Raspberry Pi OS (64-bit) Bookworm  
> **Stack:** Flask + Pyrebase4 + XGBoost + Isolation Forest

---

## Quick Start

```bash
# 1. Clone the project
git clone https://github.com/qppd/wmldad.git
cd wmldad/rpi/

# 2. Create virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Copy Firebase config
cp firebase_config.json rpi/

# 5. Set environment variables
export FIREBASE_EMAIL="esp32@your-project.iam.gserviceaccount.com"
export FIREBASE_PASSWORD="your-strong-password"
export DEVICE_ID="wm_001"

# 6. Run Flask app
python app.py

# 7. Test: Open browser to http://<rpi-ip>:5000/
```

---

## Detailed Setup

### 1. Raspberry Pi Preparation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python 3.11+ and venv
sudo apt install -y python3 python3-venv python3-pip

# Enable SSH (for headless access)
sudo systemctl enable ssh
sudo systemctl start ssh
```

### 2. Project Setup

```bash
# Clone repository
git clone https://github.com/qppd/wmldad.git
cd wmldad/rpi/

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip

# Install requirements
pip install -r requirements.txt
```

### 3. Firebase Configuration (Pyrebase4)

1. In Firebase Console → **Project Settings → General**
2. Copy the **Web API Key**, **Database URL**, **Auth Domain**, **Project ID**, **Storage Bucket**, **Messaging Sender ID**, **App ID**
3. Save as `firebase_config.json` in `rpi/` directory:

```json
{
  "apiKey": "AIzaSy...",
  "authDomain": "your-project.firebaseapp.com",
  "databaseURL": "https://your-project-default-rtdb.asia-southeast1.firebasedatabase.app",
  "projectId": "your-project",
  "storageBucket": "your-project.appspot.com",
  "messagingSenderId": "123456789",
  "appId": "1:123456789:web:abcdef123456"
}
```

```bash
# Verify file exists
ls -la firebase_config.json
```

### 4. Authentication (Email/Password)

Pyrebase4 uses email/password authentication:

```python
# In your code:
email = "esp32@your-project.iam.gserviceaccount.com"
password = "your-strong-password"
```

**Important:** The email/password user must be created in Firebase Console → **Authentication → Sign-in method → Email/Password → Add user**.

### 5. ML Model Files

Copy trained models from training phase:

```bash
# From Google Colab / Jupyter training
cp training/xgboost_leak_model.json rpi/models/
cp training/isolation_forest.pkl rpi/models/
cp training/scaler.pkl rpi/models/
```

Verify:
```bash
ls -la models/
# Should show: xgboost_leak_model.json, isolation_forest.pkl, scaler.pkl
```

---

## Running the Backend

### Development Mode

```bash
cd rpi/
source venv/bin/activate
python app.py
```

Output:
```
 * Running on all addresses (0.0.0.0:5000)
 * Running on http://192.168.1.xxx:5000
 * Running on http://127.0.0.1:5000
```

Access dashboard at: `http://<rpi-ip>:5000/`

### Production Mode (systemd Service)

Create service file:

```bash
sudo tee /etc/systemd/system/water-meter.service > /dev/null <<'EOF'
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

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable water-meter.service
sudo systemctl start water-meter.service

# Check status
sudo systemctl status water-meter.service

# View logs
journalctl -u water-meter.service -f
```

---

## Requirements.txt

```
flask>=2.3
pyrebase4>=4.5
xgboost>=2.0
scikit-learn>=1.3
pandas>=2.0
numpy>=1.24
joblib>=1.3
gunicorn>=21.0
python-dotenv>=1.0
requests>=2.31
```

---

## Project Structure (rpi/)

```
rpi/
├── app.py                 # Main Flask application
├── firebase_listener.py   # Pyrebase4 polling
├── ml_inference.py        # XGBoost + Isolation Forest inference
├── alert_engine.py        # Notification system (Telegram, Email)
├── models/                # Trained ML models
│   ├── xgboost_leak_model.json
│   ├── isolation_forest.pkl
│   └── scaler.pkl
├── templates/             # Jinja2 HTML templates
│   ├── base.html
│   ├── dashboard.html
│   └── alerts.html
├── static/                # CSS, JS, Chart.js
│   ├── css/
│   ├── js/
│   └── lib/
├── requirements.txt
├── firebase_config.json   # Firebase config for Pyrebase4 (gitignored)
├── .env                   # Environment variables (gitignored)
└── water-meter.service    # systemd service file
```

---

## Key Components

### app.py — Flask Entry Point

```python
from flask import Flask, render_template, jsonify, request
from firebase_listener import FirebaseListener
from ml_inference import LeakDetector
from alert_engine import AlertEngine
import threading
import os

app = Flask(__name__)

# Initialize components
listener = FirebaseListener(
    firebase_config_path="firebase_config.json",
    email=os.getenv("FIREBASE_EMAIL"),
    password=os.getenv("FIREBASE_PASSWORD"),
    device_id=os.getenv("DEVICE_ID", "wm_001")
)
detector = LeakDetector(
    xgb_path="models/xgboost_leak_model.json",
    iforest_path="models/isolation_forest.pkl",
    scaler_path="models/scaler.pkl"
)
alert_engine = AlertEngine(
    telegram_token=os.getenv("TELEGRAM_BOT_TOKEN"),
    telegram_chat_id=os.getenv("TELEGRAM_CHAT_ID")
)

# Start background listener
listener.start()

@app.route('/')
def dashboard():
    return render_template('dashboard.html')

@app.route('/api/latest')
def api_latest():
    latest = listener.get_latest_reading()
    return jsonify(latest)

@app.route('/api/alerts')
def api_alerts():
    alerts = listener.get_recent_alerts(limit=20)
    return jsonify(alerts)

@app.route('/api/predict', methods=['POST'])
def api_predict():
    data = request.json
    features = extract_features_from_request(data)
    result = detector.predict(features)
    return jsonify(result)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

---

### firebase_listener.py — Pyrebase4 Polling

```python
import pyrebase
import json
import threading
import time
from datetime import datetime

class FirebaseListener:
    def __init__(self, firebase_config_path, email, password, device_id):
        self.device_id = device_id
        self.last_timestamp = None
        self.email = email
        self.password = password
        
        # Load Firebase config
        with open(firebase_config_path, 'r') as f:
            config = json.load(f)
        
        # Initialize Pyrebase4
        self.firebase = pyrebase.initialize_app(config)
        self.auth = self.firebase.auth()
        self.db = self.firebase.database()
        
        # Sign in with email/password
        self.user = self.auth.sign_in_with_email_and_password(email, password)
        self.id_token = self.user['idToken']
        
        self.readings_ref = self.db.child(f"readings/{device_id}")
        self.alerts_ref = self.db.child(f"alerts/{device_id}")
        self.commands_ref = self.db.child(f"commands/{device_id}")
        
    def _refresh_token(self):
        """Refresh auth token if expired"""
        try:
            self.user = self.auth.refresh(self.user['refreshToken'])
            self.id_token = self.user['idToken']
        except Exception as e:
            print(f"Token refresh failed: {e}")
            # Re-authenticate
            self.user = self.auth.sign_in_with_email_and_password(self.email, self.password)
            self.id_token = self.user['idToken']
        
    def start(self):
        """Start polling thread"""
        self.poll_thread = threading.Thread(target=self._poll_loop, daemon=True)
        self.poll_thread.start()
        
    def _poll_loop(self):
        while True:
            try:
                self._check_new_readings()
            except Exception as e:
                print(f"Poll error: {e}")
                # Try to refresh token on auth errors
                if "permission" in str(e).lower() or "unauthorized" in str(e).lower():
                    self._refresh_token()
            time.sleep(5)  # Poll every 5 seconds
            
    def _check_new_readings(self):
        # Get latest reading
        readings = self.readings_ref.order_by_key().limit_to_last(1).get(self.id_token)
        if readings and readings.val():
            for ts, data in readings.val().items():
                if ts != self.last_timestamp:
                    self.last_timestamp = ts
                    self.process_reading(data)
                    
    def process_reading(self, data):
        # Extract features and run ML inference
        features = self.extract_features(data)
        result = detector.predict(features)
        
        if result['final'] != 'normal':
            # Write alert to Firebase
            alert_data = {
                "alert_type": result['final'],
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "confidence": result.get('confidence', 0),
                "fixture_index": data.get('fixture_index', -1),
                "action": "monitoring"
            }
            self.alerts_ref.push(alert_data, self.id_token)
            
            # Send notification
            alert_engine.send_telegram(alert_data)
            
    def get_latest_reading(self):
        readings = self.readings_ref.order_by_key().limit_to_last(1).get(self.id_token)
        return readings.val() if readings else None
        
    def get_recent_alerts(self, limit=20):
        alerts = self.alerts_ref.order_by_key().limit_to_last(limit).get(self.id_token)
        return alerts.val() if alerts else None
        
    def send_command(self, command):
        self.commands_ref.push({
            "command": command,
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "source": "dashboard"
        }, self.id_token)
```

---

### ml_inference.py — XGBoost + Isolation Forest

```python
import xgboost as xgb
import joblib
import numpy as np

class LeakDetector:
    def __init__(self, xgb_path, iforest_path, scaler_path):
        self.xgb = xgb.XGBClassifier()
        self.xgb.load_model(xgb_path)
        self.iforest = joblib.load(iforest_path)
        self.scaler = joblib.load(scaler_path)
        self.confidence_threshold = 0.80
        self.model_loaded = True
        self.n_features = 9
        
    def predict(self, features_raw):
        # Scale features
        features = self.scaler.transform(features_raw.reshape(1, -1))
        
        # 1. XGBoost prediction
        xgb_probs = self.xgb.predict_proba(features)[0]
        xgb_class = np.argmax(xgb_probs)
        xgb_confidence = xgb_probs[xgb_class]
        
        # 2. Isolation Forest anomaly score
        iforest_score = self.iforest.score_samples(features)[0]
        is_anomaly = self.iforest.predict(features)[0] == -1
        
        result = {
            'xgboost': {
                'class': ['normal', 'minor_leak', 'major_leak'][xgb_class],
                'confidence': float(xgb_confidence),
                'probabilities': {
                    'normal': float(xgb_probs[0]),
                    'minor_leak': float(xgb_probs[1]),
                    'major_leak': float(xgb_probs[2])
                }
            },
            'isolation_forest': {
                'anomaly': bool(is_anomaly),
                'score': float(iforest_score)
            }
        }
        
        # Decision logic
        if xgb_confidence >= self.confidence_threshold:
            result['final'] = result['xgboost']['class']
        elif is_anomaly:
            result['final'] = 'anomaly'
        else:
            result['final'] = 'uncertain'
            
        return result
```

---

### alert_engine.py — Notifications

```python
import requests
import smtplib
from email.mime.text import MIMEText

class AlertEngine:
    def __init__(self, telegram_token=None, telegram_chat_id=None):
        self.telegram_token = telegram_token
        self.telegram_chat_id = telegram_chat_id
        
    def send_telegram(self, alert_data):
        if not self.telegram_token or not self.telegram_chat_id:
            return
            
        message = f"""
🚨 *Water Meter Alert*
*Type:* {alert_data['alert_type']}
*Confidence:* {alert_data.get('confidence', 0):.2f}
*Fixture:* {alert_data.get('fixture_name', 'Unknown')}
*Time:* {alert_data['timestamp']}
        """
        
        url = f"https://api.telegram.org/bot{self.telegram_token}/sendMessage"
        data = {
            "chat_id": self.telegram_chat_id,
            "text": message,
            "parse_mode": "Markdown"
        }
        requests.post(url, data=data)
        
    def send_email(self, alert_data, to_email):
        # Configure with your SMTP settings
        pass
```

---

## Telegram Bot Setup

1. Open Telegram, search for **@BotFather**
2. Send `/newbot` → follow prompts
3. Save the **API token**
4. Get your **Chat ID**:
   - Send a message to your bot
   - Visit: `https://api.telegram.org/bot<TOKEN>/getUpdates`
   - Find `"chat":{"id":123456789}`
5. Add to environment:
   ```bash
   export TELEGRAM_BOT_TOKEN="your_token_here"
   export TELEGRAM_CHAT_ID="123456789"
   ```

---

## Remote Access: Port Forwarding + DDNS

### 1. Router Port Forwarding

| Setting | Value |
|---------|-------|
| **External Port** | 8443 (or any unused port) |
| **Internal IP** | Raspberry Pi's local IP (e.g., 192.168.1.100) |
| **Internal Port** | 5000 |
| **Protocol** | TCP |

**Example (TP-Link/Asus/Netgear):**
1. Access router admin (usually 192.168.1.1 or 192.168.0.1)
2. Go to **Advanced → NAT Forwarding → Port Forwarding / Virtual Server**
3. Add rule:
   - Service Name: `WaterMeter`
   - External Port: `8443`
   - Internal IP: `<rpi-local-ip>`
   - Internal Port: `5000`
   - Protocol: `TCP`
4. Save and apply

### 2. Static DHCP Reservation (Recommended)

Reserve a fixed IP for the RPi in router's DHCP settings so port forwarding doesn't break after reboot.

### 3. Dynamic DNS (Optional but Recommended)

If your ISP doesn't provide a static public IP:

**Option A: DuckDNS (Free)**
```bash
# Create account at duckdns.org
# Get token, then:
curl "https://www.duckdns.org/update?domains=yourdomain&token=yourtoken&ip="
# Add to cron for auto-update:
crontab -e
# */5 * * * * curl "https://www.duckdns.org/update?domains=yourdomain&token=yourtoken&ip="
```

**Option B: Cloudflare Tunnel (Best for HTTPS)**
```bash
# Install cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared-linux-arm64.deb

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create water-meter

# Route DNS
cloudflared tunnel route dns water-meter yourdomain.com

# Run as service
sudo cloudflared service install
```

### 4. Access from Anywhere

- **Local:** `http://<rpi-local-ip>:5000/`
- **Remote (Port Forward):** `http://<your-public-ip>:8443/`
- **Remote (DDNS + Port Forward):** `http://yourdomain.duckdns.org:8443/`
- **Remote (Cloudflare Tunnel):** `https://yourdomain.com/`

---

## Security Considerations

| Measure | Implementation |
|---------|----------------|
| **HTTPS** | Use Cloudflare Tunnel (free SSL) or Caddy/Nginx reverse proxy with Let's Encrypt |
| **Authentication** | Add basic auth or Flask-Login to dashboard |
| **Firewall** | `sudo ufw allow 5000/tcp` (local), block unnecessary ports |
| **Fail2Ban** | `sudo apt install fail2ban` — protect against brute force |
| **Change default port** | Use non-standard external port (e.g., 8443 instead of 5000) |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| **App not loading** | Check Flask output: `journalctl -u water-meter.service -f` |
| **"Internal Server Error"** | View Flask error log: `sudo journalctl -u water-meter.service --since "5 min ago"` |
| **Module not found** | Activate venv → `pip install -r requirements.txt` |
| **Memory error** | RPi 4 has 2-8GB RAM — check `free -h`. Reduce `n_estimators` in XGBoost if needed. |
| **RPi not reachable** | Check network: `ping <rpi-ip>`. Ensure port 5000 is not blocked by firewall |
| **RPi auto-start not working** | Check systemd: `sudo systemctl status water-meter.service` |
| **SD card corruption** | Use a UPS and `sudo raspi-config` → Performance → Overlay File System for read-only root |

---

## Backup & Maintenance

```bash
# Backup models
tar -czf models_backup_$(date +%Y%m%d).tar.gz models/

# Backup Firebase data (via CLI)
firebase database:get /readings > backup_readings.json

# Check disk space
df -h

# Update system
sudo apt update && sudo apt upgrade -y
```