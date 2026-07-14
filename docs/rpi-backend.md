# RPi Backend Setup Guide — Create Flask + ML Backend on Raspberry Pi

> **Hardware:** Raspberry Pi 3B+/4/5  
> **OS:** Raspberry Pi OS Trixie 64-bit (Debian 13)  
> **Stack:** Flask + Pyrebase4 + XGBoost + Isolation Forest + systemd  
> **Target:** Create all backend files from scratch

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Setup](#system-setup)
3. [Project Structure Creation](#project-structure-creation)
4. [Python Environment & Dependencies](#python-environment--dependencies)
5. [Firebase Configuration](#firebase-configuration)
6. [Create Application Files](#create-application-files)
7. [ML Model Files](#ml-model-files)
8. [Systemd Service (Auto-start)](#systemd-service-auto-start)
9. [Remote Access (Optional)](#remote-access-optional)
10. [Verification & Testing](#verification--testing)
11. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Raspberry Pi 3B+ / 4 / 5 with **Raspberry Pi OS Trixie 64-bit** installed
- SSH access enabled (see [raspberry-pi-installation.md](./raspberry-pi-installation.md))
- Firebase project created (see [firebase-setup-guide.md](./firebase-setup-guide.md))
- ESP32 firmware uploaded and sending data (see [esp32-setup-guide.md](./esp32-setup-guide.md))
- Internet connection on Pi

---

## System Setup

```bash
# 1. Update system
sudo apt update && sudo apt full-upgrade -y

# 2. Install Python 3.12+ and tools
sudo apt install -y python3 python3-venv python3-pip git

# 3. Verify Python version (should be 3.12+ on Trixie)
python3 --version
# Python 3.12.x

# 4. Enable SSH (if not already)
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

## Project Structure Creation

```bash
# 1. Clone the project (or create structure manually)
cd /home/pi
git clone https://github.com/qppd/wmldad.git
cd wmldad

# 2. Create rpi/ directory structure
mkdir -p rpi/{models,templates,static/css,static/js,static/lib}

# 3. Verify structure
tree rpi/
# rpi/
# ├── models/
# ├── templates/
# ├── static/
# │   ├── css/
# │   ├── js/
# │   └── lib/
```

---

## Python Environment & Dependencies

### 1. Create Virtual Environment

```bash
cd /home/pi/wmldad/rpi
python3 -m venv venv
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip

# Verify venv active
which python
# Should show: /home/pi/wmldad/rpi/venv/bin/python
```

### 2. Create requirements.txt

Create `rpi/requirements.txt`:

```bash
cat > requirements.txt << 'EOF'
flask>=3.0
pyrebase4>=4.5
xgboost>=2.0
scikit-learn>=1.3
pandas>=2.0
numpy>=1.24
joblib>=1.3
gunicorn>=21.0
python-dotenv>=1.0
requests>=2.31
EOF
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt

# Verify key packages
python -c "import flask, pyrebase, xgboost, sklearn, pandas, numpy; print('All packages OK')"
```

> **Note:** On Raspberry Pi (ARM64), `xgboost` and `numpy` compile from source — this may take 5-10 minutes. Use `pip install --no-cache-dir -r requirements.txt` if you have disk space constraints.

---

## Firebase Configuration

### 1. Get Firebase Web Config

1. Open [Firebase Console](https://console.firebase.google.com)
2. Select your project
3. **Project Settings** (gear icon) → **General** tab
4. Scroll to **Your apps** → Click **Web app** (</>) or create one
5. Copy the `firebaseConfig` object

### 2. Create firebase_config.json

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

> **Important:** Replace all values with your actual Firebase config. This file is gitignored.

### 3. Create .env File

```bash
cat > .env << 'EOF'
FIREBASE_EMAIL=esp32@your-project.iam.gserviceaccount.com
FIREBASE_PASSWORD=your-strong-password
DEVICE_ID=wm_001
FLASK_HOST=0.0.0.0
FLASK_PORT=5000
EOF
```

> **Note:** The email/password must match a user created in Firebase Console → **Authentication** → **Sign-in method** → **Email/Password** → **Add user**.

---

## Create Application Files

### 1. Create `rpi/app.py` — Main Flask Application

```bash
cat > app.py << 'EOF'
"""
Water Meter Leak Detection — Flask Backend
Runs on Raspberry Pi, polls Firebase, runs ML inference, serves dashboard.
"""

import os
import json
import logging
from flask import Flask, render_template, jsonify, request
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Local imports
from firebase_listener import FirebaseListener
from ml_inference import LeakDetector
from alert_engine import AlertEngine

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Flask app
app = Flask(__name__)

# Configuration from environment
FIREBASE_CONFIG_PATH = os.getenv('FIREBASE_CONFIG_PATH', 'firebase_config.json')
FIREBASE_EMAIL = os.getenv('FIREBASE_EMAIL')
FIREBASE_PASSWORD = os.getenv('FIREBASE_PASSWORD')
DEVICE_ID = os.getenv('DEVICE_ID', 'wm_001')

# ML model paths
XGB_PATH = 'models/xgboost_leak_model.json'
IFOREST_PATH = 'models/isolation_forest.pkl'
SCALER_PATH = 'models/scaler.pkl'

# Global components
firebase_listener = None
detector = None
alert_engine = None


def initialize_components():
    """Initialize all backend components."""
    global firebase_listener, detector, alert_engine
    
    logger.info("Initializing ML detector...")
    detector = LeakDetector(
        xgb_path=XGB_PATH,
        iforest_path=IFOREST_PATH,
        scaler_path=SCALER_PATH
    )
    detector.warm_up()
    
    logger.info("Initializing alert engine...")
    alert_engine = AlertEngine()
    
    logger.info("Initializing Firebase listener...")
    firebase_listener = FirebaseListener(
        config_path=FIREBASE_CONFIG_PATH,
        email=FIREBASE_EMAIL,
        password=FIREBASE_PASSWORD,
        device_id=DEVICE_ID,
        poll_interval=5
    )
    firebase_listener.set_detector(detector)
    firebase_listener.set_alert_engine(alert_engine)
    firebase_listener.start()
    
    logger.info("All components initialized successfully")


# Initialize on startup
initialize_components()


# ==================== ROUTES ====================

@app.route('/')
def dashboard():
    """Main dashboard page."""
    return render_template('dashboard.html')


@app.route('/api/health')
def health():
    """Health check endpoint."""
    return jsonify({
        'status': 'healthy',
        'firebase_connected': firebase_listener.is_connected() if firebase_listener else False,
        'model_loaded': detector.model_loaded if detector else False,
        'device_id': DEVICE_ID
    })


@app.route('/api/latest')
def api_latest():
    """Get latest sensor reading from Firebase."""
    if not firebase_listener:
        return jsonify({'error': 'Firebase listener not initialized'}), 503
    
    latest = firebase_listener.get_latest_reading()
    return jsonify(latest) if latest else jsonify({})


@app.route('/api/alerts')
def api_alerts():
    """Get recent alerts from Firebase."""
    if not firebase_listener:
        return jsonify({'error': 'Firebase listener not initialized'}), 503
    
    alerts = firebase_listener.get_recent_alerts(limit=50)
    return jsonify(alerts) if alerts else jsonify({})


@app.route('/api/predict', methods=['POST'])
def api_predict():
    """Run ML inference on provided features."""
    if not detector or not detector.model_loaded:
        return jsonify({'error': 'ML model not loaded'}), 503
    
    data = request.get_json()
    if not data or 'features' not in data:
        return jsonify({'error': 'Missing features in request'}), 400
    
    try:
        import numpy as np
        features = np.array(data['features'], dtype=np.float32).reshape(1, -1)
        result = detector.predict(features)
        return jsonify(result)
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        return jsonify({'error': str(e)}), 500


@app.route('/api/command', methods=['POST'])
def api_command():
    """Send command to ESP32 via Firebase."""
    if not firebase_listener:
        return jsonify({'error': 'Firebase listener not initialized'}), 503
    
    data = request.get_json()
    if not data or 'command' not in data:
        return jsonify({'error': 'Missing command'}), 400
    
    try:
        firebase_listener.send_command(data['command'])
        return jsonify({'status': 'sent', 'command': data['command']})
    except Exception as e:
        logger.error(f"Command send error: {e}")
        return jsonify({'error': str(e)}), 500


# ==================== MAIN ====================

if __name__ == '__main__':
    port = int(os.getenv('FLASK_PORT', 5000))
    host = os.getenv('FLASK_HOST', '0.0.0.0')
    
    logger.info(f"Starting Flask on {host}:{port}")
    app.run(host=host, port=port, debug=False, threaded=True)
EOF
```

---

### 2. Create `rpi/firebase_listener.py` — Pyrebase4 Polling

```bash
cat > firebase_listener.py << 'EOF'
"""
Firebase Listener — Polls Firebase Realtime DB for new sensor readings
using Pyrebase4 (Email/Password auth).
"""

import pyrebase
import json
import threading
import time
import logging
from datetime import datetime
from typing import Optional, Callable, Dict, Any

logger = logging.getLogger(__name__)


class FirebaseListener:
    """Polls Firebase for new readings, runs ML inference, writes alerts."""
    
    def __init__(
        self,
        config_path: str,
        email: str,
        password: str,
        device_id: str,
        poll_interval: int = 5
    ):
        self.device_id = device_id
        self.poll_interval = poll_interval
        self.email = email
        self.password = password
        self.last_timestamp = None
        self.running = False
        self._detector = None
        self._alert_engine = None
        self._poll_thread: Optional[threading.Thread] = None
        
        # Load Firebase config
        with open(config_path, 'r') as f:
            self.firebase_config = json.load(f)
        
        # Initialize Pyrebase4
        self.firebase = pyrebase.initialize_app(self.firebase_config)
        self.auth = self.firebase.auth()
        self.db = self.firebase.database()
        
        # Sign in
        self._sign_in()
        
        # Database references
        self.readings_ref = self.db.child(f"readings/{device_id}")
        self.alerts_ref = self.db.child(f"alerts/{device_id}")
        self.commands_ref = self.db.child(f"commands/{device_id}")
        self.device_ref = self.db.child(f"devices/{device_id}")
    
    def _sign_in(self):
        """Authenticate with Firebase."""
        try:
            self.user = self.auth.sign_in_with_email_and_password(
                self.email, self.password
            )
            self.id_token = self.user['idToken']
            self.refresh_token = self.user['refreshToken']
            logger.info("Firebase authentication successful")
        except Exception as e:
            logger.error(f"Firebase auth failed: {e}")
            raise
    
    def _refresh_token(self):
        """Refresh expired auth token."""
        try:
            self.user = self.auth.refresh(self.refresh_token)
            self.id_token = self.user['idToken']
            logger.info("Token refreshed")
        except Exception as e:
            logger.warning(f"Token refresh failed, re-authenticating: {e}")
            self._sign_in()
    
    def set_detector(self, detector):
        """Set ML detector for inference."""
        self._detector = detector
    
    def set_alert_engine(self, alert_engine):
        """Set alert engine for notifications."""
        self._alert_engine = alert_engine
    
    def start(self):
        """Start polling thread."""
        if self.running:
            return
        self.running = True
        self._poll_thread = threading.Thread(target=self._poll_loop, daemon=True)
        self._poll_thread.start()
        logger.info("Firebase listener started")
    
    def stop(self):
        """Stop polling thread."""
        self.running = False
        if self._poll_thread:
            self._poll_thread.join(timeout=5)
        logger.info("Firebase listener stopped")
    
    def _poll_loop(self):
        """Main polling loop."""
        while self.running:
            try:
                self._check_new_readings()
            except Exception as e:
                logger.error(f"Poll error: {e}")
                if "permission" in str(e).lower() or "unauthorized" in str(e).lower():
                    self._refresh_token()
            time.sleep(self.poll_interval)
    
    def _check_new_readings(self):
        """Fetch latest reading from Firebase."""
        readings = self.readings_ref.order_by_key().limit_to_last(1).get(self.id_token)
        
        if readings and readings.val():
            for ts, data in readings.val().items():
                if ts != self.last_timestamp:
                    self.last_timestamp = ts
                    self.process_reading(data, ts)
    
    def process_reading(self, data: Dict, timestamp: str):
        """Extract features, run ML inference, write alerts."""
        if not self._detector:
            return
        
        try:
            inlet = data.get('inlet', {})
            
            # Process each fixture (1-3)
            for fixture_idx in [1, 2, 3]:
                fixture_key = f'fixture_{fixture_idx}'
                fixture = data.get(fixture_key, {})
                
                # Only process if water is flowing
                if fixture.get('flow_rate', 0) > 0.01:
                    features = self._extract_features(data, fixture_idx, fixture, inlet)
                    result = self._detector.predict(features)
                    
                    if result['final'] != 'normal':
                        self._write_alert(result, fixture_idx, fixture, inlet, timestamp)
                        
        except Exception as e:
            logger.error(f"Error processing reading: {e}")
    
    def _extract_features(self, data: Dict, fixture_idx: int, fixture: Dict, inlet: Dict):
        """Extract 9 features from raw sensor data."""
        import numpy as np
        from datetime import datetime
        
        flow_rate = fixture.get('flow_rate', 0)
        volume = fixture.get('volume', 0)
        inlet_rate = inlet.get('flow_rate', 0)
        
        # 1. Flow rate (L/min)
        # 2. Duration (approximate from volume/rate)
        duration = volume / max(flow_rate / 60, 0.01) if flow_rate > 0 else 0
        
        # 3-4. Time features
        now = datetime.now()
        hour = now.hour
        day = now.weekday()
        
        # 5. Fixture ID
        fixture_id = fixture_idx
        
        # 6. Inlet ratio
        inlet_ratio = inlet_rate / max(flow_rate, 0.01)
        
        # 7. Rate variance (simplified)
        rate_variance = 0
        
        # 8. Night flag
        is_night = 1 if (hour >= 22 or hour < 5) else 0
        
        # 9. Pulse trend (simplified)
        pulse_trend = 0
        
        return np.array([[
            flow_rate, duration, hour, day, fixture_id,
            inlet_ratio, rate_variance, is_night, pulse_trend
        ]], dtype=np.float32)
    
    def _write_alert(self, result: Dict, fixture_idx: int, fixture: Dict, inlet: Dict, timestamp: str):
        """Write alert to Firebase /alerts."""
        fixture_names = {1: 'bidet', 2: 'kitchen', 3: 'bathroom_shower'}
        
        alert_data = {
            'alert_type': result['final'],
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'confidence': result.get('confidence', 0),
            'fixture_index': fixture_idx,
            'fixture_name': fixture_names.get(fixture_idx),
            'action': 'monitoring',
            'details': {
                'flow_rate': fixture.get('flow_rate', 0),
                'inlet_flow_rate': inlet.get('flow_rate', 0),
                'xgboost_class': result['xgboost']['class'],
                'xgboost_confidence': result['xgboost']['confidence'],
                'isolation_forest_anomaly': result['isolation_forest']['anomaly'],
                'isolation_forest_score': result['isolation_forest']['score']
            }
        }
        
        try:
            self.alerts_ref.push(alert_data, self.id_token)
            logger.warning(f"ALERT: {result['final']} on fixture {fixture_idx} (conf: {result.get('confidence', 0):.2f})")
            
            # Send notification
            if self._alert_engine:
                self._alert_engine.send_notification(alert_data)
        except Exception as e:
            logger.error(f"Failed to write alert: {e}")
    
    def get_latest_reading(self):
        """Get latest reading for API."""
        readings = self.readings_ref.order_by_key().limit_to_last(1).get(self.id_token)
        return readings.val() if readings else None
    
    def get_recent_alerts(self, limit=20):
        """Get recent alerts for API."""
        alerts = self.alerts_ref.order_by_key().limit_to_last(limit).get(self.id_token)
        return alerts.val() if alerts else None
    
    def send_command(self, command: str):
        """Send command to ESP32 via Firebase /commands."""
        self.commands_ref.push({
            'command': command,
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'source': 'dashboard'
        }, self.id_token)
    
    def is_connected(self):
        """Check Firebase connectivity."""
        try:
            self.db.child('.info/connected').get(self.id_token)
            return True
        except:
            return False
EOF
```

---

### 3. Create `rpi/ml_inference.py` — XGBoost + Isolation Forest

```bash
cat > ml_inference.py << 'EOF'
"""
ML Inference — XGBoost Classifier + Isolation Forest Anomaly Detection
"""

import xgboost as xgb
import joblib
import numpy as np
import logging

logger = logging.getLogger(__name__)


class LeakDetector:
    """Combined XGBoost + Isolation Forest leak detector."""
    
    def __init__(
        self,
        xgb_path: str,
        iforest_path: str,
        scaler_path: str,
        threshold_path: str = None
    ):
        self.xgb = xgb.XGBClassifier()
        self.xgb.load_model(xgb_path)
        self.iforest = joblib.load(iforest_path)
        self.scaler = joblib.load(scaler_path)
        self.confidence_threshold = 0.80
        self.model_loaded = True
        self.n_features = 9
        
        logger.info("ML models loaded successfully")
    
    def warm_up(self):
        """Run a dummy prediction to warm up models."""
        dummy = np.zeros((1, self.n_features), dtype=np.float32)
        try:
            self.predict(dummy)
            logger.info("Models warmed up")
        except Exception as e:
            logger.error(f"Warm-up failed: {e}")
    
    def predict(self, features_raw: np.ndarray) -> Dict:
        """
        Run inference on raw features (1x9 array).
        Returns combined XGBoost + Isolation Forest result.
        """
        # Scale features
        features = self.scaler.transform(features_raw.reshape(1, -1))
        
        # 1. XGBoost prediction
        xgb_probs = self.xgb.predict_proba(features)[0]
        xgb_class = int(np.argmax(xgb_probs))
        xgb_confidence = float(xgb_probs[xgb_class])
        
        class_names = ['normal', 'minor_leak', 'major_leak']
        
        # 2. Isolation Forest anomaly score
        iforest_score = float(self.iforest.score_samples(features)[0])
        is_anomaly = bool(self.iforest.predict(features)[0] == -1)
        
        result = {
            'xgboost': {
                'class': class_names[xgb_class],
                'confidence': xgb_confidence,
                'probabilities': {
                    'normal': float(xgb_probs[0]),
                    'minor_leak': float(xgb_probs[1]),
                    'major_leak': float(xgb_probs[2])
                }
            },
            'isolation_forest': {
                'anomaly': is_anomaly,
                'score': iforest_score
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
EOF
```

---

### 4. Create `rpi/alert_engine.py` — In-App Notifications

```bash
cat > alert_engine.py << 'EOF'
"""
Alert Engine — In-app notifications via Firebase /alerts
The web dashboard polls /alerts and displays alerts in real-time.
"""

import logging
from typing import Dict, Any

logger = logging.getLogger(__name__)


class AlertEngine:
    """Handles in-app alert notifications."""
    
    def __init__(self):
        # In-app notifications are handled by writing to Firebase /alerts
        # The Flask dashboard (JavaScript) polls /api/alerts and displays them
        # No external dependencies (email, SMS, push) required
        pass
    
    def send_notification(self, alert_data: Dict[str, Any]):
        """
        Send notification for alert.
        
        The alert is already written to Firebase /alerts by firebase_listener.
        This method is a hook for future extensions (email, webhook, etc.).
        """
        logger.info(f"Notification sent for: {alert_data['alert_type']} on {alert_data.get('fixture_name')}")
        
        # Future: add email, webhook, Telegram, etc.
        # Example:
        # if alert_data['alert_type'] in ['major_leak']:
        #     self._send_webhook(alert_data)
    
    def _send_webhook(self, alert_data: Dict):
        """Optional: send to external webhook."""
        pass
EOF
```

---

### 5. Create `rpi/templates/dashboard.html` — Main Dashboard

```bash
cat > templates/dashboard.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Water Meter Dashboard</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="{{ url_for('static', filename='css/dashboard.css') }}">
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4"></script>
</head>
<body>
    <div class="container-fluid p-3">
        <!-- Header -->
        <div class="row mb-3">
            <div class="col-12">
                <h2 class="mb-0">💧 Water Meter Monitor</h2>
                <small class="text-muted" id="last-update">Loading...</small>
            </div>
        </div>
        
        <!-- Status Cards -->
        <div class="row mb-3" id="status-cards">
            <!-- Inlet -->
            <div class="col-6 col-md-3">
                <div class="card sensor-card inlet-card">
                    <div class="card-body text-center">
                        <h6 class="card-title">Inlet</h6>
                        <div class="sensor-value" id="inlet-flow">-- L/min</div>
                        <div class="sensor-sub">Total: <span id="inlet-total">-- L</span></div>
                    </div>
                </div>
            </div>
            
            <!-- Fixture 1: Bidet -->
            <div class="col-6 col-md-3">
                <div class="card sensor-card" id="fixture1-card">
                    <div class="card-body text-center">
                        <h6 class="card-title">Bidet</h6>
                        <div class="sensor-value" id="fix1-flow">-- L/min</div>
                        <div class="sensor-sub">Total: <span id="fix1-total">-- L</span></div>
                    </div>
                </div>
            </div>
            
            <!-- Fixture 2: Kitchen -->
            <div class="col-6 col-md-3">
                <div class="card sensor-card" id="fixture2-card">
                    <div class="card-body text-center">
                        <h6 class="card-title">Kitchen</h6>
                        <div class="sensor-value" id="fix2-flow">-- L/min</div>
                        <div class="sensor-sub">Total: <span id="fix2-total">-- L</span></div>
                    </div>
                </div>
            </div>
            
            <!-- Fixture 3: Shower -->
            <div class="col-6 col-md-3">
                <div class="card sensor-card" id="fixture3-card">
                    <div class="card-body text-center">
                        <h6 class="card-title">Shower</h6>
                        <div class="sensor-value" id="fix3-flow">-- L/min</div>
                        <div class="sensor-sub">Total: <span id="fix3-total">-- L</span></div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Charts Row -->
        <div class="row mb-3">
            <div class="col-md-8">
                <div class="card">
                    <div class="card-header">Flow Rate History (Last 50 readings)</div>
                    <div class="card-body">
                        <canvas id="flowChart" height="100"></canvas>
                    </div>
                </div>
            </div>
            <div class="col-md-4">
                <div class="card">
                    <div class="card-header">System Status</div>
                    <div class="card-body">
                        <div class="mb-2">
                            <strong>Firebase:</strong> 
                            <span class="badge bg-success" id="firebase-status">Connected</span>
                        </div>
                        <div class="mb-2">
                            <strong>ML Model:</strong> 
                            <span class="badge bg-success" id="ml-status">Loaded</span>
                        </div>
                        <div class="mb-2">
                            <strong>Device:</strong> 
                            <span id="device-id">wm_001</span>
                        </div>
                        <div class="mt-3">
                            <button class="btn btn-sm btn-outline-primary" onclick="sendCommand('calibrate')">
                                Calibrate
                            </button>
                            <button class="btn btn-sm btn-outline-danger ms-2" onclick="sendCommand('reboot')">
                                Reboot ESP32
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Alerts Table -->
        <div class="row">
            <div class="col-12">
                <div class="card">
                    <div class="card-header d-flex justify-content-between">
                        <h5 class="mb-0">Recent Alerts</h5>
                        <button class="btn btn-sm btn-outline-secondary" onclick="loadAlerts()">Refresh</button>
                    </div>
                    <div class="card-body p-0">
                        <div class="table-responsive">
                            <table class="table table-hover mb-0" id="alerts-table">
                                <thead class="table-light">
                                    <tr>
                                        <th>Time</th>
                                        <th>Type</th>
                                        <th>Fixture</th>
                                        <th>Confidence</th>
                                        <th>Details</th>
                                    </tr>
                                </thead>
                                <tbody id="alerts-body">
                                    <tr><td colspan="5" class="text-center text-muted">Loading...</td></tr>
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script src="{{ url_for('static', filename='js/dashboard.js') }}"></script>
</body>
</html>
EOF
```

---

### 6. Create `rpi/static/css/dashboard.css`

```bash
cat > static/css/dashboard.css << 'EOF'
/* Water Meter Dashboard Styles */

.sensor-card {
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    transition: transform 0.2s, box-shadow 0.2s;
}
.sensor-card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 16px rgba(0,0,0,0.15);
}

.inlet-card { border-top: 4px solid #0d6efd; }
#fixture1-card { border-top: 4px solid #198754; }
#fixture2-card { border-top: 4px solid #fd7e14; }
#fixture3-card { border-top: 4px solid #6f42c1; }

.sensor-value {
    font-size: 1.5rem;
    font-weight: 700;
    color: #212529;
}
.sensor-sub {
    font-size: 0.85rem;
    color: #6c757d;
}

.card-header {
    background: #f8f9fa;
    border-bottom: 1px solid #dee2e6;
    font-weight: 600;
}

#flowChart {
    max-height: 300px;
}

.alert-minor_leak { border-left: 4px solid #fd7e14; }
.alert-major_leak { border-left: 4px solid #dc3545; }
.alert-anomaly { border-left: 4px solid #6f42c1; }

@media (max-width: 576px) {
    .sensor-value { font-size: 1.2rem; }
    .card-header { font-size: 0.9rem; }
}
EOF
```

---

### 7. Create `rpi/static/js/dashboard.js`

```bash
cat > static/js/dashboard.js << 'EOF'
// Water Meter Dashboard JavaScript

let flowChart = null;
let updateInterval = null;

const FIXTURE_NAMES = {
    1: 'Bidet',
    2: 'Kitchen',
    3: 'Bathroom Shower'
};

const ALERT_COLORS = {
    'minor_leak': 'warning',
    'major_leak': 'danger',
    'anomaly': 'purple',
    'normal': 'success'
};

// Initialize on load
document.addEventListener('DOMContentLoaded', () => {
    initFlowChart();
    loadLatest();
    loadAlerts();
    startAutoRefresh();
});

function initFlowChart() {
    const ctx = document.getElementById('flowChart').getContext('2d');
    flowChart = new Chart(ctx, {
        type: 'line',
        data: {
            labels: [],
            datasets: [
                { label: 'Inlet', data: [], borderColor: '#0d6efd', backgroundColor: 'rgba(13,110,253,0.1)', fill: true, tension: 0.3 },
                { label: 'Bidet', data: [], borderColor: '#198754', backgroundColor: 'rgba(25,135,84,0.1)', fill: true, tension: 0.3 },
                { label: 'Kitchen', data: [], borderColor: '#fd7e14', backgroundColor: 'rgba(253,126,20,0.1)', fill: true, tension: 0.3 },
                { label: 'Shower', data: [], borderColor: '#6f42c1', backgroundColor: 'rgba(111,66,193,0.1)', fill: true, tension: 0.3 }
            ]
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: { legend: { position: 'top' } },
            scales: {
                y: { beginAtZero: true, title: { display: true, text: 'L/min' } },
                x: { display: false }
            }
        }
    });
}

async function loadLatest() {
    try {
        const res = await fetch('/api/latest');
        const data = await res.json();
        
        if (data && Object.keys(data).length > 0) {
            updateDashboard(data);
        }
    } catch (e) {
        console.error('Load latest failed:', e);
        document.getElementById('firebase-status').className = 'badge bg-danger';
        document.getElementById('firebase-status').textContent = 'Disconnected';
    }
}

function updateDashboard(data) {
    // Update last update time
    document.getElementById('last-update').textContent = 
        `Last update: ${new Date().toLocaleTimeString()}`;
    
    // Inlet
    const inlet = data.inlet || {};
    document.getElementById('inlet-flow').textContent = `${inlet.flow_rate || 0} L/min`;
    document.getElementById('inlet-total').textContent = `${inlet.total || 0} L`;
    
    // Fixtures
    for (let i = 1; i <= 3; i++) {
        const fix = data[`fixture_${i}`] || {};
        document.getElementById(`fix${i}-flow`).textContent = `${fix.flow_rate || 0} L/min`;
        document.getElementById(`fix${i}-total`).textContent = `${fix.total || 0} L`;
    }
    
    // Update chart
    updateChart(inlet, data);
}

function updateChart(inlet, data) {
    const now = new Date().toLocaleTimeString();
    const maxPoints = 50;
    
    // Add new data point
    flowChart.data.labels.push(now);
    flowChart.data.datasets[0].data.push(inlet.flow_rate || 0);
    
    for (let i = 1; i <= 3; i++) {
        const fix = data[`fixture_${i}`] || {};
        flowChart.data.datasets[i].data.push(fix.flow_rate || 0);
    }
    
    // Trim old points
    if (flowChart.data.labels.length > maxPoints) {
        flowChart.data.labels.shift();
        flowChart.data.datasets.forEach(ds => ds.data.shift());
    }
    
    flowChart.update('none');
}

async function loadAlerts() {
    try {
        const res = await fetch('/api/alerts');
        const alerts = await res.json();
        renderAlerts(alerts);
    } catch (e) {
        console.error('Load alerts failed:', e);
    }
}

function renderAlerts(alerts) {
    const tbody = document.getElementById('alerts-body');
    if (!alerts || Object.keys(alerts).length === 0) {
        tbody.innerHTML = '<tr><td colspan="5" class="text-center text-muted">No alerts</td></tr>';
        return;
    }
    
    // Sort by timestamp descending
    const sorted = Object.entries(alerts)
        .sort((a, b) => b[1].timestamp.localeCompare(a[1].timestamp))
        .slice(0, 20);
    
    tbody.innerHTML = sorted.map(([key, alert]) => {
        const time = new Date(alert.timestamp).toLocaleTimeString();
        const type = alert.alert_type || 'unknown';
        const fixture = alert.fixture_name || `Fixture ${alert.fixture_index}`;
        const conf = alert.confidence ? `${(alert.confidence * 100).toFixed(1)}%` : '--';
        const color = ALERT_COLORS[type] || 'secondary';
        
        return `
            <tr class="alert-${type}">
                <td>${time}</td>
                <td><span class="badge bg-${color}">${type.replace('_', ' ')}</span></td>
                <td>${fixture}</td>
                <td>${conf}</td>
                <td>
                    <small class="text-muted">
                        Flow: ${alert.details?.flow_rate || 0} L/min | 
                        Inlet: ${alert.details?.inlet_flow_rate || 0} L/min
                    </small>
                </td>
            </tr>
        `;
    }).join('');
}

async function sendCommand(command) {
    if (!confirm(`Send "${command}" command to ESP32?`)) return;
    
    try {
        const res = await fetch('/api/command', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ command })
        });
        const result = await res.json();
        alert(`Command sent: ${result.command}`);
    } catch (e) {
        alert('Failed to send command');
    }
}

function startAutoRefresh() {
    // Refresh every 5 seconds
    updateInterval = setInterval(() => {
        loadLatest();
        loadAlerts();
    }, 5000);
}

// Cleanup on page unload
window.addEventListener('beforeunload', () => {
    if (updateInterval) clearInterval(updateInterval);
});
EOF
```

---

## ML Model Files

### If you have trained models (from Google Colab/Jupyter):

```bash
# Copy from your training environment to rpi/models/
# Required files:
cp xgboost_leak_model.json rpi/models/
cp isolation_forest.pkl rpi/models/
cp scaler.pkl rpi/models/

# Verify
ls -la rpi/models/
```

### If you DON'T have trained models yet:

The system will work with local ESP32 rules (inlet balance, continuous flow, drip detection). ML can be added later.

```bash
# Create placeholder models (for testing dashboard only)
cd rpi/models
python3 -c "
import xgboost as xgb
import numpy as np
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
import joblib

# Dummy XGBoost
xgb_model = xgb.XGBClassifier()
xgb_model.fit(np.random.rand(100, 9), np.random.randint(0, 3, 100))
xgb_model.save_model('xgboost_leak_model.json')

# Dummy Isolation Forest
iforest = IsolationForest(contamination=0.05)
iforest.fit(np.random.rand(100, 9))
joblib.dump(iforest, 'isolation_forest.pkl')

# Dummy Scaler
scaler = StandardScaler()
scaler.fit(np.random.rand(100, 9))
joblib.dump(scaler, 'scaler.pkl')

print('Placeholder models created')
"
```

> ⚠️ **Important:** Replace with real trained models from [ml-training-guide.md](./ml-training-guide.md) for production use.

---

## Systemd Service (Auto-start)

### 1. Create Service File

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

### 2. Enable & Start

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable on boot
sudo systemctl enable water-meter.service

# Start now
sudo systemctl start water-meter.service

# Check status
sudo systemctl status water-meter.service

# View logs
journalctl -u water-meter.service -f
```

---

## Remote Access (Optional)

### Port Forwarding + DDNS

| Setting | Value |
|---------|-------|
| External Port | 8443 |
| Internal IP | Raspberry Pi LAN IP |
| Internal Port | 5000 |
| Protocol | TCP |

### Cloudflare Tunnel (Recommended - Free HTTPS)

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

# Install as service
sudo cloudflared service install
```

---

## Verification & Testing

### 1. Test Flask App Manually First

```bash
cd /home/pi/wmldad/rpi
source venv/bin/activate
python app.py
# Visit http://<rpi-ip>:5000/
```

### 2. Verify Endpoints

```bash
# Health check
curl http://localhost:5000/api/health

# Latest reading
curl http://localhost:5000/api/latest

# Alerts
curl http://localhost:5000/api/alerts
```

### 3. Check Dashboard Loads

- Open browser to `http://<rpi-ip>:5000/`
- Should show flow rate cards, chart, alerts table
- Firebase status badge should be green
- ML status badge should be green

### 4. Test ESP32 → Firebase → RPi Flow

1. Run water through sensors
2. Check Firebase Console → `/readings/wm_001/` updates every 5s
3. RPi dashboard shows live flow rates
4. Simulate leak (slow drip) → alert appears in dashboard

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| **App not loading** | Check Flask output: `journalctl -u water-meter.service -f` |
| **"Internal Server Error"** | View logs: `sudo journalctl -u water-meter.service --since "5 min ago"` |
| **Module not found** | Activate venv → `pip install -r requirements.txt` |
| **Memory error** | RPi 4 has 2-8GB — reduce `n_estimators` in XGBoost if needed |
| **RPi not reachable** | Check network: `ping <rpi-ip>`. Ensure port 5000 not blocked by firewall |
| **Auto-start fails** | Check systemd: `sudo systemctl status water-meter.service` |
| **Firebase auth fails** | Verify email/password in `.env` matches Firebase Auth user |
| **No data in dashboard** | Check ESP32 serial: `screen /dev/ttyESP32 115200` — verify JSON output |
| **ML model not loaded** | Ensure `models/*.json`, `*.pkl` exist in `rpi/models/` |

---

## Backup & Maintenance

```bash
# Backup models
tar -czf models_backup_$(date +%Y%m%d).tar.gz models/

# Backup Firebase data (requires firebase-tools)
npm install -g firebase-tools
firebase login
firebase database:get /readings > backup_readings_$(date +%Y%m%d).json

# Check disk space
df -h

# Update system
sudo apt update && sudo apt upgrade -y
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Start service | `sudo systemctl start water-meter.service` |
| Stop service | `sudo systemctl stop water-meter.service` |
| Restart service | `sudo systemctl restart water-meter.service` |
| View logs | `journalctl -u water-meter.service -f` |
| Manual run | `cd rpi && source venv/bin/activate && python app.py` |
| Update deps | `cd rpi && source venv/bin/activate && pip install -r requirements.txt` |
| Check port | `netstat -tlnp | grep 5000` |

---

*Last updated: July 2026 | Raspberry Pi OS Trixie 64-bit | Flask 3.x + Pyrebase4 + XGBoost 2.x*