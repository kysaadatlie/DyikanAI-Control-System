# DyikanAI — Smart Greenhouse Control System

**University of Central Asia · Naryn, Kyrgyzstan · 2026**  
**Authors:** Saadat Orozova 

Low-cost, open-source smart greenhouse automation system built for resource-constrained regions. Runs entirely on local hardware, no internet required. Designed and tested in Naryn, Kyrgyzstan.

---

## Repository Structure

```
DyikanAI/
├── backend/
│   ├── automation_engine.py          # Threshold-based control engine
│   ├── automation_engine_flc.py      # Mamdani Fuzzy Logic Controller
│   ├── switch_automation.sh          # Engine switcher (mutual exclusion)
│   ├── mqtt_to_influx.py             # MQTT → InfluxDB bridge
│   ├── flask_api.py                  # REST API
│   ├── filter_engine.py              # Kalman filter
│   ├── deploy_frost_detector.py      # Frost detection deployment
│   ├── mosquitto/                    # MQTT broker config
│   ├── services/                     # systemd service files
│   └── .env.example
├── mamdani-fis/
│   └── mamdani_fis.ipynb             # Mamdani FIS design & simulation notebook
├── firmware/
│   └── greenhouse_firmware_v1.3/     # Arduino Mega 2560 firmware
├── frost-detector/
│   └── deploy_frost_detector.py      # XGBoost frost detection model
└── dashboard/                        # Web dashboard (React)
```

---

## System Overview

```
Sensors (Arduino Mega 2560)
    │ MQTT telemetry
    ▼
Raspberry Pi 400
    ├── Kalman Filter         — noise reduction on sensor readings
    ├── InfluxDB 2.x          — time-series storage
    ├── Flask API             — REST interface for dashboard & manual control
    ├── Automation Engine     — threshold-based or Mamdani FLC control
    └── Frost Detector        — XGBoost ML model (early warning)
    │ MQTT commands
    ▼
Arduino Mega 2560 → Relays → Pump / Lamp / Heater / Fan
```

---

## Control Modes

| Mode | File | Description |
|------|------|-------------|
| Threshold engine | `automation_engine.py` | Rule-based ON/OFF with hysteresis bands. Default on boot. |
| Mamdani FLC | `automation_engine_flc.py` | Fuzzy inference — graduated responses based on membership functions. Primary research contribution. |
| Manual | Flask API + dashboard | Direct actuator control; automation must be disabled first. |

Switch engines:
```bash
./backend/switch_automation.sh threshold   # default
./backend/switch_automation.sh flc
./backend/switch_automation.sh status
```

---

## Actuators

| Actuator | Trigger | Arduino Pin |
|----------|---------|-------------|
| Water pump | Soil moisture < 35% | D39 |
| Phytolamp | Time schedule (06:00–22:00) | D47 |
| Heater | Air temp < 20°C | D41 |
| Fan | Air temp > 26°C or humidity > 85% | D43 |

---

## Setup

**Prerequisites:** Mosquitto (port 1883), InfluxDB 2.x (port 8086), Python 3.

```bash
git clone https://github.com/auuerk/DyikanAI.git
cd DyikanAI
python3 -m venv venv && source venv/bin/activate
pip install paho-mqtt influxdb-client python-dotenv scikit-fuzzy numpy

cp backend/.env.example backend/.env
# Edit .env — set MQTT credentials and control thresholds

sudo cp backend/services/*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now agricontrol-automation
```

**Verify:**
```bash
./backend/switch_automation.sh status
tail -f ~/iot-backend/auto.log
```

---

## Key Configuration (`.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `TEMP_HEATER_ON` | 20.0 °C | Heater ON threshold |
| `TEMP_FAN_ON` | 26.0 °C | Fan ON threshold |
| `HUMIDITY_HIGH` | 85.0 % | Fan ON (humidity) |
| `SOIL_PCT_DRY` | 35.0 % | Pump trigger |
| `PUMP_COOLDOWN` | 120 min | Min time between pump runs |
| `LIGHT_ON_HOUR` | 6 | Lamp schedule start |
| `LIGHT_OFF_HOUR` | 22 | Lamp schedule end |

---

## Project Layers

| Layer | Owner | Scope |
|-------|-------|-------|
| Edge / Backend | Aruuke | Sensors, MQTT, InfluxDB, Kalman filter, Flask API |
| Control | Saadat | Threshold engine, Mamdani FLC, manual control |
| AI / Interface | Alfiia | Frost detection (XGBoost), dashboard, AI chatbot |
