# Wi-Fi Intrusion Detection System

> A real-time, standalone Wi-Fi attack detector built on ESP32 that monitors all nearby wireless traffic, detects active cyberattacks, and alerts through visual and physical feedback — all for under ₹935.

---

## Table of Contents

- [What is this?](#what-is-this)
- [Why it exists](#why-it-exists)
- [How it works](#how-it-works)
- [Attacks detected](#attacks-detected)
- [Alert system](#alert-system)
- [Hardware](#hardware)
- [Pin connections](#pin-connections)
- [Project structure](#project-structure)
- [Setup and flashing](#setup-and-flashing)
- [Live demo](#live-demo)
- [Limitations](#limitations)
- [Future improvements](#future-improvements)

---

## What is this?

Haleem WiFi IDS is a **portable, always-on Wi-Fi Intrusion Detection System** built around the ESP32 microcontroller. It runs completely standalone — no laptop, no internet connection, no software to maintain.

It places the ESP32 radio in **promiscuous mode**, which means it can receive and analyse every Wi-Fi management frame in the air around it — including frames from devices it is not connected to. It hops across all 13 Wi-Fi channels every 500ms to ensure full spectrum coverage, detects known attack patterns in real time, and immediately alerts through:

- An **OLED display** showing attack type, severity, attacker MAC address, and timestamp
- An **RGB LED** (three separate LEDs: red, green, yellow) showing threat level by color
- A **push button** to acknowledge and clear alerts

Think of it as a **security camera for your wireless airspace**.

---

## Why it exists

Wi-Fi attacks happen at the **802.11 radio layer** — a level that is completely invisible to the devices being targeted. A laptop or phone under a deauthentication attack has no idea it is being attacked because the malicious packets are processed by the Wi-Fi hardware before the operating system ever sees them.

Common security tools — antivirus software, firewalls, VPNs — **do not protect against radio-layer attacks**. They operate one level too high. The attacks this system detects happen before a device even associates with a network.

This project fills that gap by monitoring the raw 802.11 layer where these attacks actually occur, at a hardware cost of under ₹935.

---

## How it works

```
Wi-Fi Radio (promiscuous mode, ch 1-13)
        │
        │  every packet as packet_info_t
        ▼
┌───────────────────┐    ┌───────────────────────┐
│  deauth_detector  │    │  rogue_ap_detector    │
│  (sliding window  │    │  (SSID/BSSID table    │
│   per MAC addr)   │    │   mismatch detection) │
└────────┬──────────┘    └──────────┬────────────┘
         │                          │
         └────────┬─────────────────┘
                  │  event_log_add()
                  ▼
         ┌─────────────────┐
         │   event_log     │  ← 30-event ring buffer in RAM
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────────┐
         │  correlation_engine │  ← detects attack chains
         └────────┬────────────┘
                  │
       ┌──────────┼───────────────┐
       ▼          ▼               ▼
  OLED display  RGB LED      Push button
  (rotates      (color =     (acknowledges
   events)       severity)    and clears)
```

### Channel hopping

The ESP32 switches between Wi-Fi channels 1 through 13 every 500ms. This ensures that attacks happening on any channel are detected within at most one full hop cycle (6.5 seconds for 13 channels).

### Packet processing pipeline

Every captured frame is passed as a structured `packet_info_t` record containing the frame type, source and destination MAC addresses, SSID (if present), BSSID, RSSI (signal strength), and channel number. The deauth detector and rogue AP detector both receive every packet simultaneously.

### Event log

All detected events are stored in a ring buffer of 30 records in RAM. Each record holds the attack type, severity level (0–3), source MAC address, human-readable description, and a millisecond timestamp. The OLED cycles through recent events automatically, displaying one every 3 seconds.

### Correlation engine

This is the core innovation of the project. The correlation engine watches the event log for combinations of events. If a deauth flood and a rogue AP alert both appear within the same time window (default: 30 seconds), it fires a **COORDINATED ATTACK** alert with CRITICAL severity. This pattern is the exact sequence used in real-world Wi-Fi credential harvesting attacks.

---

## Attacks detected

### 1. Deauthentication Flood

An attacker sends forged 802.11 deauthentication frames to forcibly disconnect devices from a network. Since deauth frames in 802.11 are unauthenticated (before WPA3), any device can send them with any spoofed source MAC address.

The detector tracks the number of deauth frames from each source MAC in a sliding time window. When the count crosses configurable thresholds, it logs events at increasing severity levels (LOW → MEDIUM → HIGH → CRITICAL).

Broadcast deauth attacks (where the destination is the broadcast MAC FF:FF:FF:FF:FF:FF, targeting all devices at once) are detected separately and rated at higher severity than targeted attacks.

**Real-world use:** Deauth floods are used to kick all devices off a legitimate network so they automatically reconnect — to a rogue AP controlled by the attacker.

### 2. Rogue Access Point (Evil Twin)

The detector maintains a table of every SSID (network name) it has seen, mapped to the BSSID (hardware MAC address of the access point) it first appeared with. When it sees the same SSID broadcast from a different BSSID, it raises a Rogue AP alert.

Severity is weighted by the RSSI of the rogue AP — a rogue AP with a stronger signal than the legitimate one is rated CRITICAL because it is more likely to successfully attract devices to connect.

**Real-world use:** An attacker sets up a fake hotspot with the same name as a cafe, airport, or office Wi-Fi. Users' devices automatically connect to it. The attacker then intercepts all traffic — passwords, session tokens, emails.

### 3. Coordinated Attack Chain

When both a deauth flood and a rogue AP are detected within the same time window, the correlation engine flags this as a coordinated attack. This is the most dangerous pattern because it represents an active, intentional campaign rather than noise.

**Real-world use:** This is the standard sequence in Wi-Fi credential harvesting: flood the real network with deauths → users disconnect → their devices auto-connect to the evil twin → attacker captures the WPA handshake or serves a fake login page.

---

## Alert system

Every event is classified into one of four severity levels. Each level produces a specific LED behavior so the threat level is immediately readable at a glance.

| Severity | Trigger condition | LED color | LED pattern |
|---|---|---|---|
| LOW | Few deauth frames, weak rogue signal | Green | Slow blink 1 Hz |
| MEDIUM | Moderate deauth rate, medium signal | Yellow | Solid on |
| HIGH | Heavy deauth flood, strong rogue AP | Red | Fast blink 2 Hz |
| CRITICAL | Coordinated attack chain detected | Red | Rapid blink 10 Hz |

The OLED shows:
- Current channel being monitored
- Total packet count
- Total event count
- On alert: attack type, severity, attacker MAC, SSID, and elapsed time

Pressing the push button clears all events from the log and resets the LED to solid green (monitoring state).

---

## Hardware

| # | Component | Specification | Role | Cost |
|---|---|---|---|---|
| 1 | ESP32 Development Board | WROOM-32, 38-pin | Main processor, Wi-Fi radio | ₹350 |
| 2 | OLED Display | SSD1306, I2C, 0.96", 128×64 | Live event display | ₹120 |
| 3 | Red LED | 5mm | HIGH/CRITICAL alert indicator | ₹3 |
| 4 | Green LED | 5mm | SAFE / LOW alert indicator | ₹3 |
| 5 | Yellow LED | 5mm | MEDIUM alert indicator | ₹4 |
| 6 | Resistors | 220Ω × 3 | Current limiting for LEDs | ₹5 |
| 7 | Push Button | Tactile switch | Acknowledge and clear alerts | ₹10 |
| 8 | Breadboard | 830-point full size | Component mounting | ₹80 |
| 9 | Jumper wires | Male-to-Male | Connections | ₹60 |
| — | USB cable | Micro-USB, data capable | Power + flashing | ₹80 |
| **Total** | | | | **₹715** |

---

## Pin connections

### OLED Display (I2C)

| OLED pin | ESP32 pin |
|---|---|
| VCC | 3.3V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

### LEDs (3 separate LEDs, common cathode to GND)

Each LED anode connects through a 220Ω resistor to its GPIO pin. The cathode (short leg) connects directly to GND.

| LED color | GPIO pin | Meaning |
|---|---|---|
| Red | GPIO 25 | HIGH / CRITICAL threat |
| Green | GPIO 27 | Safe / monitoring / LOW |
| Yellow | GPIO 26 | MEDIUM threat |

### Push Button

| Button leg | ESP32 pin | Notes |
|---|---|---|
| Leg 1 | GPIO 33 | Uses INPUT_PULLUP — no resistor needed |
| Leg 2 | GND | |

---

## Project structure

```
WiFi_IDS_ESP32/
├── WiFi_IDS_ESP32.ino        # Main entry point — setup() and loop()
├── platformio.ini            # PlatformIO build config
│
├── sniffer/
│   ├── sniffer.h             # Promiscuous mode Wi-Fi sniffer interface
│   └── sniffer.cpp           # Channel hopping, packet capture, callback
│
├── detection/
│   ├── deauth_detector.h/.cpp      # Deauth flood detection with sliding window
│   ├── rogue_ap_detector.h/.cpp    # Evil twin / rogue AP detection
│   └── correlation_engine.h/.cpp   # Multi-event attack chain correlation
│
├── display/
│   ├── display.h             # OLED display interface
│   └── display.cpp           # SSD1306 rendering — monitoring and alert screens
│
├── led_alert/
│   ├── led_alert.h           # RGB LED alert interface
│   └── led_alert.cpp         # Non-blocking LED blink patterns per severity
│
├── push_button/
│   ├── push_button.h         # Debounced push button interface
│   └── push_button.cpp       # Edge detection, debounce, pressed flag
│
└── utils/
    ├── event_log.h           # Event log interface
    └── event_log.cpp         # 30-event ring buffer, add/get/clear/count
```

---

## Setup and flashing

### Requirements

- [PlatformIO](https://platformio.org/) (VS Code extension or CLI)
- USB data cable (not charge-only)

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

lib_deps =
    adafruit/Adafruit SSD1306 @ ^2.5.7
    adafruit/Adafruit GFX Library @ ^1.11.9
```

### Steps

1. Clone this repository
2. Open the folder in VS Code with PlatformIO installed
3. Click **Build** (✓) — PlatformIO will download all dependencies automatically
4. Connect ESP32 via Micro-USB
5. Click **Upload** (→)
6. Open **Serial Monitor** at 115200 baud

### Expected Serial output on boot

```
========== ESP32 Wi-Fi IDS v3.0 ==========
[LED] Alert system ready.
[BTN] Push button ready on GPIO 33.
[DISPLAY] OLED ready.
[MAIN] System ready -- sniffing all channels.
[MAIN] Press button to acknowledge and clear alerts.
```

Every second you will see:
```
[MAIN] pkts=247 events=0 ch=7
[MAIN] pkts=501 events=0 ch=8
```

When an attack is detected:
```
[EVENT] DEAUTH_FLOOD | SEV:HIGH | SRC:AA:BB:CC:DD:EE:FF | count=45
[LED] Setting severity HIGH
[EVENT] ROGUE_AP | SEV:CRITICAL | SSID='HomeWifi' | rogue BSSID:11:22:33:44:55:66
[CORRELATION] Coordinated attack detected — DEAUTH + ROGUE_AP within window
```

---

## Live demo

### Demo method — Evil Twin (no special tools needed)

This demo works with any Android phone — no rooting required.

**Prepare before the demo:**

1. Power on the IDS and note any Wi-Fi network name visible nearby (your college Wi-Fi, a hotspot name, anything)
2. On your Android phone go to **Settings → Mobile Hotspot → Hotspot Name**
3. Set the hotspot name to **exactly match** the nearby network name (same capitalization, same spaces)

**During the demo:**

1. Show the OLED in normal monitoring state — green LED, channel hopping visible
2. Turn ON your phone's mobile hotspot
3. Within 3–5 seconds the IDS detects the Evil Twin:
   - OLED switches to alert screen showing `ROGUE_AP`, severity, and your phone's MAC address
   - LED changes from green to yellow or red
4. Press the push button — OLED clears, LED resets to green, system returns to monitoring

**What to tell judges:**

The IDS detected that a new access point appeared broadcasting the same network name as a known network, but with a different hardware address. In a real attack, this rogue AP would intercept the Wi-Fi credentials of any device that auto-connected to it. The system identified the attacker's MAC address, classified the threat severity, and logged the event — all within seconds of the attack starting.

---

## Limitations

**Detection only, not prevention.** This system detects and alerts. It cannot block attackers, disconnect rogue APs, or protect devices on the network. It is a sensor, not a shield.

**No persistent storage.** Events are stored in RAM (30 records). All history is lost when the ESP32 is powered off. A future version could log events to an SD card.

**False positives are possible.** Some legitimate network equipment sends deauth frames during normal operation, such as when a device roams between access points. Detection thresholds are tuned to reduce false positives but cannot eliminate them entirely.

**Cannot read encrypted traffic.** The system reads only 802.11 management frames, which are always transmitted unencrypted. It cannot inspect data frames or read the content of network communications.

**Limited range.** The ESP32 radio has a practical indoor range of roughly 20–50 metres. Attackers outside this range will not be detected.

**No WPA3 deauth protection detection.** WPA3 introduces Management Frame Protection (MFP), which authenticates deauth frames. The system currently does not distinguish between MFP-protected and unprotected deauths.

---

## Future improvements

- SD card logging for persistent event history and forensic analysis
- Web dashboard served from ESP32 SoftAP for remote monitoring from a browser
- Whitelist of known-good BSSIDs to reduce false positives
- Alert export via HTTP POST to a server or IFTTT webhook
- Battery and power bank support for portable deployment
- WPA3 Management Frame Protection awareness
- Probe request tracking to detect device fingerprinting attacks
- Speaker / buzzer integration for audio alerts

---

## Tech stack

| Layer | Technology |
|---|---|
| Hardware | ESP32 WROOM-32 |
| Framework | Arduino (via PlatformIO) |
| Build system | PlatformIO |
| Display library | Adafruit SSD1306 + Adafruit GFX |
| Wi-Fi layer | ESP-IDF promiscuous mode API (via Arduino wrapper) |
| Language | C++ (Arduino dialect) |

---

## Author

**Haleem**
Final Year Project — Cybersecurity / Embedded Systems
Built with ESP32, Arduino framework, and PlatformIO.

---

## Disclaimer

This project is built for **educational and research purposes only**. The detection techniques used are passive — the device only listens and never transmits attack frames. Always ensure you have permission before monitoring any network environment. The author is not responsible for any misuse of this project.
