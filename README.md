# 🧭 Lodestone

> A handheld GPS navigation and device-tracking system built on the ESP32.

Lodestone pairs wirelessly with other Lodestone units and a companion Progressive Web App (PWA). All communication runs over MQTT via a cloud broker — no Bluetooth or local network pairing required.

---

## 📋 Table of Contents

- [Features](#features)
- [Hardware](#hardware)
- [Wiring](#wiring)
- [Software Dependencies](#software-dependencies)
- [MQTT Broker Setup](#mqtt-broker-setup)
- [Flashing the Firmware](#flashing-the-firmware)
- [Deploying the PWA](#deploying-the-pwa)
- [How It Works](#how-it-works)
- [Navigation Guide](#navigation-guide)
- [EEPROM Layout](#eeprom-layout)
- [Compass Calibration](#compass-calibration)
- [Repository Structure](#repository-structure)
- [Contributing](#contributing)
- [License](#license)

---

## ✨ Features

### On the Device

| Feature | Description |
|---|---|
| 🧭 Direction Finder | Point-and-navigate to any saved waypoint with an animated arrow |
| 🌐 Digital Compass | Tilt-compensated heading with smoothing |
| 🚗 Speedometer | Live GPS speed, max speed, and trip distance |
| 📡 Device Tracker | Track the real-time position of any paired Lodestone |
| 🔗 Pair Lodestone | Send and accept pairing requests from nearby hardware |
| 💬 Messaging | Send/receive text messages with paired devices and app users |
| 📶 WiFi Setup | Scan and connect to networks including WPA2 Enterprise |
| 📍 Save Locations | Store up to 10 named waypoints in EEPROM |

All settings and data **persist across reboots** via EEPROM. Boot animation, sleep timeout, display brightness, and device name are all configurable.

### Companion PWA

- 📱 Installable on Android and iOS (add to home screen)
- 🗺️ Live map showing paired device positions (Leaflet / OpenStreetMap)
- 📍 Saved locations viewer and manager
- 💬 Messaging inbox and composer
- 🔗 Pair / unpair devices from the browser
- ✈️ Works fully offline after first load (service worker cached)

---

## 🔧 Hardware

| Component | Part |
|---|---|
| Microcontroller | ESP32 (30-pin DevKit) |
| Display | SH1106G 1.3″ 128×64 OLED (I2C) |
| IMU | LSM303 (accelerometer + magnetometer, I2C) |
| GPS | Any UART NMEA module (tested at 9600 baud) |
| Input | 5-way rocker / joystick (UP, DOWN, LEFT, RIGHT, PRESS) |
| Power | LiPo battery + USB charging module |

---

## 🔌 Wiring

| Signal | ESP32 GPIO |
|---|---|
| I2C SDA (OLED + LSM303) | GPIO 21 |
| I2C SCL (OLED + LSM303) | GPIO 22 |
| GPS UART RX | GPIO 16 |
| GPS UART TX | GPIO 17 |
| Button UP | GPIO 32 |
| Button DOWN | GPIO 33 |
| Button LEFT | GPIO 25 |
| Button RIGHT | GPIO 26 |
| Button PRESS (select) | GPIO 27 |

**I2C addresses:** OLED → `0x3C` | LSM303 Accelerometer → `0x19` | LSM303 Magnetometer → `0x1E`

---

## 📦 Software Dependencies

Install these libraries via the **Arduino Library Manager** before compiling:

| Library | Purpose |
|---|---|
| `Adafruit GFX Library` | Graphics primitives |
| `Adafruit SH110X` | SH1106G OLED driver |
| `TinyGPS++` | NMEA GPS parsing |
| `PubSubClient` | MQTT client |
| `ArduinoJson` | JSON serialisation |
| `EEPROM` | Built-in (ESP32 Arduino core) |
| `Wire` | Built-in I2C |
| `esp_eap_client` | WPA2 Enterprise WiFi (built-in ESP-IDF) |

**Board:** ESP32 Dev Module

Add the ESP32 Arduino core via **Boards Manager** using this URL:
```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

---

## ☁️ MQTT Broker Setup

Lodestone uses **HiveMQ Cloud** (free tier) as the broker. All traffic is TLS-encrypted on port **8883**.

1. Create a free account at [hivemq.com/mqtt-cloud-broker](https://www.hivemq.com/mqtt-cloud-broker/)
2. Create a cluster and note the hostname
3. Create credentials (username + password)
4. Update the firmware defines:

```cpp
#define MQTT_HOST   "your-cluster-id.s1.eu.hivemq.cloud"
#define MQTT_PORT   8883
#define MQTT_USER   "your-username"
#define MQTT_PASS   "your-password"
```

5. Update the same credentials in `js/app.js` (`mqttHost`, `mqttUser`, `mqttPass`)

> ⚠️ **Security:** Never commit real credentials to a public repository.
> - For the firmware, use `#include "secrets.h"` and add it to `.gitignore`
> - For the PWA, use a `.env` file or build-time substitution

---

## 🚀 Flashing the Firmware

1. Clone this repository
2. Open `Lodestone_Final_vXX_app.ino` in **Arduino IDE 2.x**
3. Install all libraries listed above
4. Select board: **ESP32 Dev Module**
5. Set upload speed: **921600**
6. Select the correct COM port
7. Click **Upload**

On first boot, the device shows the boot animation and prompts you to set up WiFi from the Settings menu.

---

## 🌐 Deploying the PWA

The `LodestoneApp/` folder is a self-contained PWA. Host it on any static web server. **HTTPS is required** for service workers and PWA install prompts.

### GitHub Pages (recommended)

1. Push `LodestoneApp/` contents to a `gh-pages` branch (or configure Pages to serve from `/docs`)
2. Visit `https://<your-username>.github.io/<repo-name>/`
3. On mobile: tap browser menu → **"Add to Home Screen"**

### Local Testing

```bash
cd LodestoneApp
npx serve .
# or
python3 -m http.server 8080
```

---

## ⚙️ How It Works

All communication flows through the MQTT broker. Devices and the app never talk directly to each other.

```
Lodestone A ──┐
Lodestone B ──┼──► HiveMQ Cloud (MQTT) ◄── PWA App
Lodestone C ──┘
```

### MQTT Topic Reference

| Topic | Direction | Purpose |
|---|---|---|
| `lodestone/announce` | Everyone → Everyone | Presence beacon (name, MAC, type) |
| `lodestone/locations` | Device → App | Share saved waypoints on connect |
| `lodestone/request/<mac>` | Any → Device | Initiate pairing handshake |
| `lodestone/accept/<mac>` | Device → Requester | Accept pair request |
| `lodestone/pair/<receiver>_<sender>` | Device → Paired device | Live GPS position for tracking |
| `lodestone/devices/<mac>` | Device → App | Live GPS position for map |
| `lodestone/msg/<mac>` | Any → Device/App | Targeted message delivery |
| `lodestone/unpair/<mac>` | Any → Target | Tear down pairing on both sides |
| `lodestone/loc/request/<mac>` | App → Device | Request saved locations |
| `lodestone/loc/response/<mac>` | Device → App | Deliver saved locations |

### Pairing Handshake

```
Device A                         Device B
   │                                │
   ├── request/<B_mac> ────────────►│  Shows "PAIR?" dialog
   │                                │  User presses PRESS
   │◄── accept/<A_mac> ─────────────┤  Saves A to EEPROM
   │  Saves B to EEPROM             │
   └── Both subscribe to pair topics ┘
```

Pairing is stored in EEPROM and survives reboots. On reconnect, devices automatically resubscribe to each other's pair topics.

### Position Updates

Each Lodestone publishes GPS position every **5 seconds** (when a fix is valid) to:
- `lodestone/devices/<mac>` — for the app's live map
- `lodestone/pair/<peerMac>_<myMac>` — for each paired device's tracking screen

Unpaired devices never receive position data.

### Messaging

Every device subscribes to `lodestone/msg/<myMac>` on connect. "Send to all" iterates the paired MAC list and delivers to each peer individually — there is no global broadcast topic. Messages from unknown senders are silently dropped.

---

## 🕹️ Navigation Guide

### Main Menu

```
┌─────────────────────┐
│  DIRECTION FINDER   │  → Navigate to a saved waypoint
│  COMPASS            │  → Digital compass with heading
│  SPEEDOMETER        │  → Speed, max speed, trip distance
│  DEVICE TRACKER     │  → Track / pair / message Lodestones
│  SETTINGS           │  → WiFi, name, display, sleep timeout
└─────────────────────┘
```

### Button Reference

| Button | Action |
|---|---|
| UP / DOWN | Scroll menu |
| LEFT | Back |
| RIGHT | Context action (e.g. unpair in tracker list) |
| PRESS | Select / confirm |

### Device Tracker Sub-menu

```
TRACK LODESTONE   →  Navigate toward a paired Lodestone
PAIR LODESTONE    →  Send pair request to nearby devices
CONNECT APP       →  Pair with the PWA companion app
MESSAGES (N)      →  Inbox and composer
```

---

## 💾 EEPROM Layout

| Address | Size | Contents |
|---|---|---|
| 0 | 340 bytes | Saved locations (10 × 34 bytes) |
| 340 | 760 bytes | Saved WiFi networks (5 × 152 bytes) |
| 1100 | 100 bytes | Settings (device name, sleep timeout, etc.) |
| 1200 | 200 bytes | Paired MACs (10 × 20 bytes) |

**Total used:** 1,400 bytes of a 2,048-byte allocation.

---

## 🔄 Compass Calibration

The LSM303 magnetometer requires **hard-iron calibration** for accurate readings.

1. From the Settings menu, select **Calibrate Compass**
2. Slowly rotate the device through all orientations for ~30 seconds
3. Min/max values are saved to EEPROM and applied automatically on every boot

---

## 📁 Repository Structure

```
Lodestone/
├── Lodestone_Final_vXX_app.ino   # ESP32 firmware (Arduino sketch)
├── LodestoneApp/
│   ├── index.html                # PWA shell and all CSS
│   ├── js/
│   │   └── app.js                # All PWA logic (MQTT, map, UI)
│   ├── sw.js                     # Service worker (offline caching)
│   ├── manifest.json             # PWA manifest
│   └── icons/
│       ├── icon-192.png
│       └── icon-512.png
└── README.md
```

---

## 🤝 Contributing

Contributions are welcome! To get started:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m "Add your feature"`
4. Push to your branch: `git push origin feature/your-feature`
5. Open a Pull Request

Please keep PRs focused and include a clear description of what was changed and why.

---

## 📄 License

MIT — do whatever you want with it, no warranty implied.

---

## 🙏 Credits

Built with:

- [TinyGPS++](http://arduiniana.org/libraries/tinygpsplus/) — NMEA GPS parsing
- [PubSubClient](https://github.com/knolleary/pubsubclient) — MQTT client
- [ArduinoJson](https://arduinojson.org/) — JSON serialisation
- [Adafruit GFX](https://github.com/adafruit/Adafruit-GFX-Library) — Graphics primitives
- [Leaflet](https://leafletjs.com/) — Interactive maps
- [MQTT.js](https://github.com/mqttjs/MQTT.js) — Browser MQTT client
