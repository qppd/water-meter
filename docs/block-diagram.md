# Block Diagram — Water Meter with Leak Detection (ESP32 → Firebase → RPi)

## System Block Diagram

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
block-beta
    columns 7
    
    block:plumbing:7
        columns 7
        
        Inlet["Main Water<br/>Supply"]:1
        FS_In["Inlet Flow<br/>Sensor"]:1
        CV_In["Check<br/>Valve"]:1
        Joint["Junction"]:1
        Fixtures["To Fixtures<br/>1–3"]:3
    end
    
    space:7
    
    block:fixtures:7
        columns 7
        F1["Fixture 1<br/>Sensor"]:1
        F2["Fixture 2<br/>Sensor"]:1
        F3["Fixture 3<br/>Sensor"]:1
        CV1["Check<br/>Vlv"] CV2["Check<br/>Vlv"] CV3["Check<br/>Vlv"]
    end
    
    space:7
    
    block:esp32:7
        columns 7
        
        block:pins:7
            columns 7
            P26["GPIO 26"] P25["GPIO 25"] P33["GPIO 33"] P32["GPIO 32"] Pins_Spacer:1 Pins_Spacer2:1 Pins_Spacer3:1
        end
        
        block:mcu:7
            columns 7
            MCU["ESP32 38-Pin<br/>CP2102<br/>Xtensa LX6<br/>WiFi + BLE"]:7
        end
        
        P26 --> MCU
        P25 --> MCU
        P33 --> MCU
        P32 --> MCU
    end
    
    space:7
    
    block:cloud:7
        columns 7
        Firebase[" Firebase<br/>Realtime DB"]:3
        RPi[" Raspberry Pi<br/>Flask + ML"]:4
    end
    
    FS_In --> P26
    F1 --> P25
    F2 --> P33
    F3 --> P32
    
    MCU --> Firebase
    Firebase --> RPi
```

</details>

---

## Pin Connections (ESP32 38-Pin with Expansion Board)

| Component | ESP32 Pin | Expansion Board | Notes |
|-----------|-----------|-----------------|-------|
| **Flow Sensor 1 — Inlet** | GPIO 26 | Screw terminal 1 | Direct connection, no pull-up needed |
| **Flow Sensor 2 — Fixture 1 (Bidet)** | GPIO 25 | Screw terminal 2 | Direct connection |
| **Flow Sensor 3 — Fixture 2 (Kitchen)** | GPIO 33 | Screw terminal 3 | Direct connection |
| **Flow Sensor 4 — Fixture 3 (Bathroom Shower)** | GPIO 32 | Screw terminal 4 | Direct connection |

---

## Wiring Diagram (Simplified)

```
ESP32 38-Pin Expansion Board
┌─────────────────────────────────────────────────────┐
│  [26] ──────┬── YF-S201 Inlet (Yellow)              │
│  [25] ──────┬── YF-S201 Fixture 1 (Yellow)          │
│  [33] ──────┬── YF-S201 Fixture 2 (Yellow)          │
│  [32] ──────┬── YF-S201 Fixture 3 (Yellow)          │
│                                                     │
│  5V  ──────┬── YF-S201 VCC (Red wires)             │
│  GND ──────┬── All sensor GND (Black wires)        │
└─────────────────────────────────────────────────────┘
```

---

## Sensor Wiring (YF-S201)

```
YF-S201 Flow Sensor
┌──────────────┐
│              │
│  Red   ─────┼──── 5V (VIN from ESP32/Expansion Board)
│  Black ─────┼──── GND
│  Yellow ────┼──── GPIO (26, 25, 33, 32) — direct connection
│              │
│  [Flow →]    │   ← Arrow indicates water flow direction
└──────────────┘
```

> **Important:** The arrow on the sensor body MUST point in the direction of water flow. Installing it backwards will give no readings.

---

## Power Distribution

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
block-beta
    columns 5
    
    AC["220V AC<br/>Outlet"]:1
    
    PSU["5V 2A<br/>USB Adapter"]:2
    
    ESPV["ESP32 VIN<br/>(5V)"]:1
    
    SensorV["Flow Sensors<br/>VCC (5V)"]:1
    
    AC --> PSU
    PSU --> ESPV
    PSU --> SensorV
```

</details>

> **Note:** The 12V solenoid valve power supply and LM2596 step-down regulator have been removed from this configuration. Check valves provide backflow prevention; automatic shutoff via solenoid valves is not included in this version.

---

## Component Layout (Enclosure)

> Mermaid-based diagram (SVG export removed; source below)

<details>
<summary><b> Mermaid Source</b> (click to expand)</summary>

```mermaid
block-beta
    columns 4
    
    block:enclosure:4
        columns 4
        
        ESP32Board["ESP32 +<br/>Expansion Board"]:1
        
        Terminal["Terminal Block<br/>Sensor Inputs<br/>(4 sensors)"]:2
        
        PSU2["5V Adapter<br/>(mounted)"]:2
    end
```

</details>

> **Enclosure:** Use an ABS project box (200×120×70mm) with cable glands for waterproof sensor cable entry.

---

## Pinout Reference (ESP32 38-Pin)

```
                   ┌─────────────┐
             EN ──┤ 1         38├── VBAT
           GPIO36─┤ 2         37├── GPIO23
           GPIO39─┤ 3         36├── GPIO22
           GPIO34─┤ 4  E   P  35├── TXD0
           GPIO35─┤ 5  S   3  34├── RXD0
           GPIO32─┤ 6   P   2  33├── GPIO21
           GPIO33─┤ 7   3   1  32├── GPIO19
           GPIO25─┤ 8   8      31├── GPIO18
           GPIO26─┤ 9          30├── GPIO5
           GPIO27─┤10          29├── GPIO17 (TXD2)
           GPIO14─┤11          28├── GPIO16 (RXD2)
           GPIO12─┤12          27├── GPIO4
           GPIO13─┤13          26├── GPIO0 (BOOT)
              GND ─┤14          25├── GPIO2 (LED)
           GPIO15─┤15          24├── GPIO15
           ───────┤16          23├── ───────
              3.3V ─┤17          22├── ───────
              5V  ─┤18          21├── ───────
              GND ─┤19          20├── ───────
                   └─────────────┘
```

> Flow sensors on **GPIO 26, 25, 33, 32** — direct connection, no pull-up resistors needed.