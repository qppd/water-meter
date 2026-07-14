# Flowchart — Water Meter with Leak Detection (ESP32 → Firebase → RPi Backend)

## 1. Main System Flow (High-Level)

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart TD
    Start((Start)) --> Init[ESP32 Initialization]
    Init --> Sensors[Initialize 4 Flow Sensors & Attach ISRs]
    Sensors --> WiFi[Connect to WiFi]
    WiFi --> Conn{Connected?}
    Conn -->|Yes| FirebaseInit[Initialize Firebase-ESP-Client]
    Conn -->|No| Offline[Offline Mode - SPIFFS Logging]
    FirebaseInit --> Stream[Start Firebase Stream Listener]
    Offline --> MainLoop[Enter Main Loop]
    Stream --> MainLoop
    
    MainLoop --> ReadPulses[Read All Pulse Counters]
    ReadPulses --> CalcFlow[Calculate Flow Metrics per Fixture]
    CalcFlow --> UpdateLED[Update Status LEDs]
    UpdateLED --> LocalRules[Apply Local Leak Rules]
    
    LocalRules --> Interval{Upload Interval?}
    Interval -->|Yes| PushData[Push to Firebase]
    Interval -->|No| CmdCheck{Command Received?}
    
    PushData --> PushOK{Success?}
    PushOK -->|Yes| ClearBuf[Clear Local Buffer]
    PushOK -->|No| QueueSPIFFS[Queue to SPIFFS]
    
    ClearBuf --> CmdCheck
    QueueSPIFFS --> CmdCheck
    
    CmdCheck -->|Yes| ExecCmd[Execute Command]
    CmdCheck -->|No| MainLoop
    
    ExecCmd -->|Calibrate| CalMode[Enter Calibration Mode]
    ExecCmd -->|Reboot| Reboot[Reboot ESP32]
    CalMode --> MainLoop
    Reboot --> Start
```

</details>

---

## 2. Firebase Data Flow (ESP32 → Firebase → RPi)

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart LR
    subgraph ESP32["ESP32 (Firebase-ESP-Client)"]
        ESP_Read[/Read Sensors/] --> ESP_Build[Build JSON Payload]
        ESP_Build --> ESP_Push[(Push to Firebase)]
        ESP_Stream[/Stream Commands/] --> ESP_Cmd{New Command?}
        ESP_Cmd -->|calibrate| ESP_Cal[Enter Calibration]
        ESP_Cmd -->|reboot| ESP_Reboot[Reboot ESP32]
    end
    
    subgraph Firebase["Firebase Realtime DB"]
        ESP_Push --> FB_Readings[(/readings)]
        FB_Commands[(/commands)] --> ESP_Stream
        FB_Alerts[(/alerts)] --> FB_Models[(/models)]
    end
    
    subgraph RPi["RPi Backend (Pyrebase4)"]
        RPi_Poll[/Poll Listener/] --> FB_Readings
        RPi_Poll --> RPi_Features[Extract Features]
        RPi_Features --> RPi_XGB[XGBoost Inference]
        RPi_Features --> RPi_IF[Isolation Forest]
        RPi_XGB --> RPi_Leak{Leak Detected?}
        RPi_IF --> RPi_Leak
        RPi_Leak -->|Yes| RPi_Alert[Write Alert]
        RPi_Leak -->|No| RPi_Log[Log Normal]
        RPi_Alert --> FB_Alerts
        RPi_Alert --> RPi_Notify[In-App Notification]
    end
    
    subgraph User["User Interface"]
        User_Dash[/Web Dashboard/] --> FB_Models
        User_Dash --> FB_Readings
        User_Alert[/In-App Alert/] --> RPi_Notify
        User_Cmd[/User Command/] --> FB_Commands
    end
```

</details>

---

## 3. ESP32 ISR Pulse Processing

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart TD
    Pulse[/Pulse from Flow Sensor/] --> ISR[ISR Triggered]
    ISR --> Time[Read millis]
    Time --> Debounce{Debounce Check dt > 5ms?}
    Debounce -->|Yes| Count[Increment Pulse Counter]
    Debounce -->|No| Ignore[Ignore - Bounce]
    Count --> Update[Update Last Pulse Time]
    Ignore --> Return[Return to Main Loop]
    Update --> Return
```

</details>

---

## 4. RPi Feature Extraction Pipeline

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart TD
    Raw[/Raw Firebase Data/] --> Parse[Parse JSON]
    Parse --> Loop{For Each Fixture}
    Loop -->|Fixture 1-3| Extract[Extract Raw Metrics]
    Extract --> Compute[Compute Features]
    
    Compute --> F1[flow_rate L/min]
    Compute --> F2[duration_seconds]
    Compute --> F3[hour_of_day]
    Compute --> F4[day_of_week]
    Compute --> F5[fixture_id]
    Compute --> F6[inlet_fixture_ratio]
    Compute --> F7[rate_variance_10s]
    Compute --> F8[is_night_time]
    Compute --> F9[pulse_trend]
    
    F1 --> Vector[Feature Vector - 9 features]
    F2 --> Vector
    F3 --> Vector
    F4 --> Vector
    F5 --> Vector
    F6 --> Vector
    F7 --> Vector
    F8 --> Vector
    F9 --> Vector
    
    Vector --> Scale[Scale & Normalize]
    Scale --> Model[ML Models - XGBoost + Isolation Forest]
```

</details>

---

## 5. ML Inference & Decision Flow

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart TD
    Features[/Feature Vector - 9 features/] --> XGB[XGBoost Predict]
    XGB --> Probs[Class Probabilities]
    Probs --> Argmax{argmax Class}
    
    Argmax -->|normal conf GT 0.80| Normal[Normal Usage]
    Argmax -->|minor_leak conf GT 0.70| Minor[Minor Leak]
    Argmax -->|major_leak conf GT 0.85| Major[Major Leak]
    Argmax -->|low confidence LT 0.70| Uncertain[Uncertain]
    
    Uncertain --> IF[Isolation Forest Anomaly Score]
    IF --> IF_Thresh{Score GT Threshold?}
    IF_Thresh -->|Yes| Anomaly[Anomaly Detected]
    IF_Thresh -->|No| Wait[Wait for More Data]
    
    Minor --> MinorCount{Consecutive GE 3?}
    MinorCount -->|Yes| ConfirmedMinor[Confirmed Minor Leak]
    MinorCount -->|No| WatchMinor[Increment Counter & Watch]
    
    Major --> ConfirmedMajor[Confirmed Major Leak]
    Anomaly --> Alert[Write Alert to Firebase]
    ConfirmedMinor --> Alert
    ConfirmedMajor --> Alert
    
    Alert --> Notify[In-App Notification]
    Alert --> Cmd[Write Command to Firebase]
```

</details>

---

## 6. Firebase Command Execution (ESP32)

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart TD
    Stream[/Firebase Stream Event commands/device_id/] --> CmdType{Command Type?}
    
    CmdType -->|calibrate| CalStart[Start Calibration Routine]
    CmdType -->|reboot| Reboot[Reboot ESP32]
    CmdType -->|reset| Reset[Reset Counters]
    CmdType -->|config| Config[Update Config]
    
    CalStart --> CalStatus[Update Status & LED]
    Reboot --> CalStatus
    Reset --> CalStatus
    Config --> CalStatus
    
    CalStatus --> Ack[Send Acknowledgment command_acknowledged]
```

</details>

---

## 7. Local Leak Detection Rules (ESP32 Fallback)

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart TD
    Cycle[Every Read Cycle] --> Rule1[Rule 1: Hidden Leak]
    Rule1 --> Check1{Inlet Volume GT Sum Fixtures + 10%?}
    Check1 -->|Yes| Alert1[Hidden Leak Alert]
    Check1 -->|No| OK1[Balance OK]
    
    Cycle --> Rule2[Rule 2: Continuous Flow]
    Rule2 --> Loop{For Each Fixture}
    Loop --> Check2{Pulse GT 0 for GT 30 min?}
    Check2 -->|Yes| Alert2[Stuck Valve / Running Toilet]
    Check2 -->|No| OK2[Fixture OK]
    
    Cycle --> Rule3[Rule 3: Drip Detection]
    Loop2{For Each Fixture} --> Check3{Flow 0.1-0.5 L/min for GT 5 min?}
    Check3 -->|Yes| Alert3[Drip Leak Suspected]
    Check3 -->|No| OK3[No Drip]
    
    Alert1 --> FirebaseAlert[Send Firebase Alert]
    Alert2 --> FirebaseAlert
    Alert3 --> FirebaseAlert
```

</details>

---

## 8. Full System Data Flow

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
flowchart LR
    %% Physical Layer
    Water[/Water Flow/]:::physical --> Sensor[YF-S201 Flow Sensor]:::physical
    Sensor --> Pulse[/Pulse Signal/]:::physical
    
    %% Firmware Layer
    Pulse --> ISR[ISR Pulse Counter]:::firmware
    ISR --> Debounce[Debounce 5ms]:::firmware
    Debounce --> Calc[Calculate Flow & Volume]:::firmware
    Calc --> LocalRules[Local Leak Rules]:::firmware
    Calc --> FirebaseClient[Firebase-ESP-Client]:::firmware
    LocalRules --> FirebaseClient
    
    %% Cloud Layer
    FirebaseClient -->|HTTPS SSE| Firebase[(Firebase Realtime DB)]:::cloud
    
    %% Backend Layer
    Firebase -->|Poll| Pyrebase[/Pyrebase4 Poll/]:::backend
    Pyrebase --> Features[Feature Extraction]:::backend
    Features --> XGB[XGBoost Inference]:::ml
    Features --> IF[Isolation Forest Anomaly]:::ml
    XGB --> AlertEngine[Alert Engine]:::backend
    IF --> AlertEngine
    AlertEngine --> Notify[In-App Notification]:::user
    
    %% User Layer
    Firebase --> Dashboard[Web Dashboard]:::user
    Firebase -->|Stream| ESP32Cmd[ESP32 Command Stream]:::firmware
    ESP32Cmd --> Handler[Command Handler]:::firmware
    
    classDef physical fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    classDef firmware fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef cloud fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef backend fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef ml fill:#fce4ec,stroke:#c62828,stroke-width:2px
    classDef user fill:#fffde7,stroke:#f9a825,stroke-width:2px
```

</details>