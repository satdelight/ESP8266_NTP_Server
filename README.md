# ESP8266 GPS-backed NTP Server (NEO-6M, no PPS) + Web Status

A small **NTP server** based on an **ESP8266 D1 mini (AZ-Delivery)** that gets its time from a **u-blox NEO-6M** GPS module and provides it to your local network via **NTP (UDP/123)**. It also exposes a simple **HTTP status page** so you can verify GPS/NTP without needing a USB serial connection.

> Note: Without a **1PPS** (pulse-per-second) signal, accuracy is limited. Typical offsets are in the range of a few 10–100 ms (Wi‑Fi jitter + NMEA latency).

---

## Features

- NTP server on **UDP port 123**
- GPS time from **NMEA RMC** (date/time) and **NMEA GGA** (fix/sats/HDOP)
- GPS output configured via **UBX**: **only RMC + GGA enabled**, other NMEA sentences disabled
- **Web status page** at `http://<ESP-IP>/` showing:
  - `haveTime` / GPS UTC date & time
  - Fix status, satellites, HDOP
  - Last received NMEA lines (Raw / RMC / GGA)

---

## Hardware

- **AZ-Delivery D1 mini (ESP8266)**
- **NEO-6M GPS module** (e.g. GY-NEO6MV2, pins labeled `VCC RX TX GND`)
- Optional external GPS antenna (often improves reception)
- USB power supply / power bank (useful if the GPS must sit by a window)

---

## Wiring

### D1 mini ↔ NEO-6M (module pins: `VCC RX TX GND`)

| NEO-6M Pin | D1 mini Pin | Notes |
|---|---|---|
| VCC | 5V *(or 3V3)* | Many GY-NEO6MV2 boards include an LDO and accept 5V |
| GND | G / GND | common ground |
| TX | D5 (GPIO14) | **GPS TX → ESP RX** |
| RX | D6 (GPIO12) | **GPS RX ← ESP TX** |

**Important:** TX/RX must be **crossed**.

---

## Software / Setup

### Arduino IDE
1. Install the **ESP8266 board package** (if you haven’t already)  
   Additional Boards Manager URL:  
   `http://arduino.esp8266.com/stable/package_esp8266com_index.json`
2. Select the board:  
   **Tools → Board → ESP8266 Boards → “LOLIN(WEMOS) D1 R2 & mini”**
3. If upload issues occur, set **Upload Speed** to **115200**
4. Install libraries (Arduino Library Manager):
   - **TinyGPSPlus**

### Wi‑Fi credentials
Edit in the sketch:
```cpp
const char* WIFI_SSID = "...";
const char* WIFI_PASS = "...";
```

---

## Usage

1. Flash the sketch.
2. Place the ESP + GPS where reception is good (often near a window) and power it.
3. Find the ESP IP address in your router’s client list (or via Serial Monitor at 115200).
4. Open the status page:  
   `http://<ESP-IP>/`

When everything works you should see:
- `haveTime: YES`
- `gps.time valid: YES`
- `gps.date valid: YES`
- satellites > 0
- last `RMC`/`GGA` lines present

---

## Testing NTP on Windows (GUI)

Example: **NetTime** (standalone GUI)
- Add your ESP IP (e.g. `192.168.1.18`) as a time server
- For testing, disable/remove other servers so only the ESP is used
- Click **Update Now**
- Status should become `Good` and show offset/lag

---

## Troubleshooting

### Status page shows `haveTime: NO` and no NMEA lines
- TX/RX swapped (most common) → **GPS TX → D5**, **GPS RX → D6**
- Wrong baud rate (usually 9600, sometimes 38400)
- Check power (5V vs 3V3)
- No GPS reception indoors → move closer to a window / outside

### NTP client cannot reach the server
- Ensure `haveTime: YES` (server replies only when time is valid)
- Check router/AP “client isolation” settings
- Check Windows firewall as a quick test

---

## Accuracy / Limitations

- Without PPS the time is derived from NMEA sentences, which adds latency/jitter.
- Wi‑Fi adds additional variable delay.
- For best accuracy you need a GPS module with **1PPS** connected to an interrupt pin and proper time discipline (not part of this project).

---

## License

MIT
