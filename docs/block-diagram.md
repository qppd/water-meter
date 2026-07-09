# Block Diagram — Water Meter Hardware

## Hardware Block Diagram

```mermaid
block-beta
    columns 5
    
    block:psu:3
        PSU["Power Supply Unit"]
    end
    
    space:2
    
    block:sensor:3
        FlowSensor["Water Flow Sensor<br/>Hall-Effect Pulse Output"]
    end
    
    space:2
    
    block:esp32:5
        columns 3
        
        block:power:1
            VReg["Voltage Regulator<br/>3.3V"]
        end
        
        block:mcu:2
            ESP32C["ESP32<br/>Xtensa LX6<br/>WiFi + BLE"]
        end
        
        block:io:2
            GPIO["GPIO
            ─────────
            Pulse Input (GPIO)
            LED Indicator
            Button / Reset"]
        end
    end
    
    block:connectivity:2
        WiFi["WiFi 2.4GHz<br/>802.11 b/g/n"]
    end
    
    space
    
    block:storage:2
        Flash["Flash Memory<br/>SPIFFS"]
    end
    
    space
    
    block:display:2
        Optional["Optional OLED<br/>Display (I2C)"]
    end
    
    PSU --> VReg
    FlowSensor --> GPIO
    VReg --> ESP32C
    ESP32C --> WiFi
    ESP32C --> Flash
    ESP32C --> GPIO
    ESP32C --> Optional
```

## Pin Connections

| Component       | ESP32 Pin | Notes                        |
|-----------------|-----------|------------------------------|
| Flow Sensor Out | GPIO 34   | Pulse input (rising edge)    |
| LED Indicator   | GPIO 2    | Onboard LED / status         |
| Button (Reset)  | EN        | Pull-up to 3.3V              |
| OLED SDA        | GPIO 21   | (Optional) I2C data          |
| OLED SCL        | GPIO 22   | (Optional) I2C clock         |
| VCC (Sensor)    | 5V / 3.3V | Per sensor spec              |
| GND             | GND       | Common ground                |

## Power Supply

```mermaid
block-beta
    columns 4
    
    AC["220V AC"] --> AC2DC["AC-DC<br/>Adapter<br/>5V / 2A"]
    AC2DC --> ESPPWR["ESP32<br/>VIN (5V)"]
    AC2DC --> SensorPWR["Flow Sensor<br/>VCC"]
    AC2DC --> VReg3V3["3.3V Regulator"]
    VReg3V3 --> ESP3V3["ESP32<br/>3.3V Rail"]
```

## Bill of Materials (Suggested)

| Item                    | Quantity | Notes                  |
|-------------------------|----------|------------------------|
| ESP32 Dev Board         | 1        | Any variant            |
| Water Flow Sensor       | 1        | YF-S201 or similar     |
| 5V / 2A Power Adapter  | 1        | USB or barrel jack     |
| Jumper Wires            | Several  | Female-to-female       |
| Resistors (10kΩ)        | 1-2      | Pull-up if needed      |
| OLED 128x64 (Optional)  | 1        | I2C display            |
