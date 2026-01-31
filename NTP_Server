/*
  Project: ESP8266 GPS-backed NTP Server (no PPS) + HTTP Status Page
  Board:   AZ-Delivery D1 mini (ESP8266)
  GPS:     u-blox NEO-6M (NMEA + UBX config)

  Purpose:
    Provide NTP time on the local network using UTC time received from GPS.
    Since this setup has NO 1PPS signal, accuracy is limited (typically tens of ms).

  Wiring (D1 mini <-> NEO-6M):
    - GPS VCC  -> 5V   (or 3V3 if your breakout requires it)
    - GPS GND  -> G / GND
    - GPS TXD  -> D5 (GPIO14)   // ESP receives NMEA (SoftwareSerial RX)
    - GPS RXD  -> D6 (GPIO12)   // ESP sends UBX config (SoftwareSerial TX)

    Note: TX/RX are crossed (GPS TX -> ESP RX, GPS RX -> ESP TX).

  Serial / Debug:
    - USB Serial (Serial) at 115200 baud for logs (optional)
    - GPS uses SoftwareSerial at 9600 baud

  GPS configuration at startup (via UBX-CFG-MSG):
    - Enable:  RMC, GGA (UART)
    - Disable: GLL, GSA, GSV, VTG (UART)
    This reduces serial traffic and improves time-stamp stability.

  HTTP status:
    - A small status page is available at: http://<ESP-IP>/
    - Shows: haveTime, GPS UTC, satellites, HDOP, last raw NMEA line, last RMC, last GGA

  Libraries (Arduino IDE Library Manager):
    - TinyGPSPlus (Mikal Hart)
    - ESP8266WiFi (built-in with ESP8266 core)
    - WiFiUdp (built-in)
    - ESP8266WebServer (built-in with ESP8266 core)
    - SoftwareSerial (built-in)

  NTP:
    - UDP port 123
    - Minimal RFC5905-like response
    - Stratum set to 2 (honest-ish without PPS discipline)

  Limitations / Notes:
    - Indoors reception may be poor; first fix can take minutes.
    - Without PPS, time-jitter is normal; WiFi adds additional jitter.
*/

#include <ESP8266WiFi.h>
#include <WiFiUdp.h>
#include <ESP8266WebServer.h>
#include <TinyGPSPlus.h>
#include <SoftwareSerial.h>

// -------- WiFi config --------
const char* WIFI_SSID = "YOUR_SSID";
const char* WIFI_PASS = "YOUR_PASSWORD";

// -------- GPS (NEO-6M) --------
static const uint8_t GPS_RX_PIN = D5; // ESP RX  <- GPS TX
static const uint8_t GPS_TX_PIN = D6; // ESP TX  -> GPS RX
static const uint32_t GPS_BAUD  = 9600;

// -------- NTP --------
static const uint16_t NTP_PORT = 123;
static const uint32_t NTP_EPOCH_OFFSET = 2208988800UL; // Unix(1970) -> NTP(1900)

WiFiUDP udp;
WiFiUDP dummy; // not used; keeps some IDEs quiet in certain configs

ESP8266WebServer web(80);

TinyGPSPlus gps;
SoftwareSerial gpsSerial(GPS_RX_PIN, GPS_TX_PIN); // RX, TX

bool haveTime = false;
uint32_t lastUnixTime = 0;     // seconds since 1970-01-01 UTC
uint32_t lastSyncMillis = 0;   // millis() when lastUnixTime was captured

// --- debug: last received NMEA lines (for remote troubleshooting via HTTP) ---
String lastNmeaLine;
String lastRmcLine;
String lastGgaLine;

// -------------------- Helpers --------------------
String ipToString(const IPAddress& ip) {
  return String(ip[0]) + "." + String(ip[1]) + "." + String(ip[2]) + "." + String(ip[3]);
}

// -------------------- Time conversion helpers --------------------
static bool isLeap(int y) {
  return ((y % 4 == 0) && (y % 100 != 0)) || (y % 400 == 0);
}
static uint32_t daysBeforeYear(int y) {
  uint32_t days = 0;
  for (int year = 1970; year < y; year++) days += isLeap(year) ? 366 : 365;
  return days;
}
static uint16_t daysBeforeMonth(int y, int m) {
  static const uint16_t cumDaysNorm[] = {0,31,59,90,120,151,181,212,243,273,304,334};
  uint16_t d = cumDaysNorm[m - 1];
  if (m > 2 && isLeap(y)) d += 1;
  return d;
}
static uint32_t toUnix(int year, int month, int day, int hour, int minute, int second) {
  uint32_t days = daysBeforeYear(year) + daysBeforeMonth(year, month) + (day - 1);
  return days * 86400UL + (uint32_t)hour * 3600UL + (uint32_t)minute * 60UL + (uint32_t)second;
}

// -------------------- UBX helpers --------------------
// Send UBX message (header+payload) and append checksum
void sendUBX(const uint8_t *msg, uint16_t len) {
  uint8_t ckA = 0, ckB = 0;
  // checksum over class..payload (starting at msg[2])
  for (uint16_t i = 2; i < len; i++) {
    ckA = ckA + msg[i];
    ckB = ckB + ckA;
  }
  for (uint16_t i = 0; i < len; i++) gpsSerial.write(msg[i]);
  gpsSerial.write(ckA);
  gpsSerial.write(ckB);
  gpsSerial.flush();
}

void setNMEARate(uint8_t msgClass, uint8_t msgId, uint8_t rateUART) {
  // UBX-CFG-MSG (0x06 0x01), length 8
  uint8_t ubx[] = {
    0xB5, 0x62, 0x06, 0x01,
    0x08, 0x00,
    msgClass, msgId,
    0x00,       // I2C rate
    rateUART,   // UART rate
    0x00,       // USB rate
    0x00        // SPI rate / reserved
  };
  sendUBX(ubx, sizeof(ubx));
  delay(60);
}

void configureGPS_RMC_GGA_only() {
  // Keep these ON:
  setNMEARate(0xF0, 0x04, 1); // RMC on (UTC time+date)
  setNMEARate(0xF0, 0x00, 1); // GGA on (fix quality, sats, alt)

  // Turn the rest OFF:
  setNMEARate(0xF0, 0x01, 0); // GLL off
  setNMEARate(0xF0, 0x02, 0); // GSA off
  setNMEARate(0xF0, 0x03, 0); // GSV off
  setNMEARate(0xF0, 0x05, 0); // VTG off
}

void saveGPSConfig() {
  // UBX-CFG-CFG (0x06 0x09), length 13
  const uint8_t ubx[] = {
    0xB5, 0x62, 0x06, 0x09,
    0x0D, 0x00,
    0x00, 0x00, 0x00, 0x00,  // clearMask
    0xFF, 0xFF, 0x00, 0x00,  // saveMask (common subset)
    0x00, 0x00, 0x00, 0x00,  // loadMask
    0x17                     // deviceMask (try all)
  };
  sendUBX(ubx, sizeof(ubx));
  delay(200);
}

// -------------------- HTTP status page --------------------
void handleRoot() {
  String html;
  html.reserve(2500);

  html += "<!doctype html><html><head><meta charset='utf-8'/>";
  html += "<meta name='viewport' content='width=device-width, initial-scale=1'/>";
  html += "<title>ESP8266 GPS NTP Status</title>";
  html += "<style>body{font-family:sans-serif;max-width:900px;margin:20px;}pre{background:#f4f4f4;padding:10px;border-radius:6px;}</style>";
  html += "</head><body>";

  html += "<h2>ESP8266 GPS NTP Status</h2>";
  html += "<p><b>WiFi IP:</b> " + ipToString(WiFi.localIP()) + "</p>";
  html += "<p><b>NTP:</b> UDP/123</p>";

  html += "<h3>Time</h3><ul>";
  html += "<li>haveTime: <b>" + String(haveTime ? "YES" : "NO") + "</b></li>";
  html += "<li>gps.time valid: " + String(gps.time.isValid() ? "YES" : "NO") + "</li>";
  html += "<li>gps.date valid: " + String(gps.date.isValid() ? "YES" : "NO") + "</li>";
  if (gps.date.isValid() && gps.time.isValid()) {
    char buf[64];
    snprintf(buf, sizeof(buf), "%04d-%02d-%02d %02d:%02d:%02d UTC",
             gps.date.year(), gps.date.month(), gps.date.day(),
             gps.time.hour(), gps.time.minute(), gps.time.second());
    html += String("<li>GPS UTC: ") + buf + "</li>";
  }
  html += "</ul>";

  html += "<h3>Fix</h3><ul>";
  html += "<li>location valid: " + String(gps.location.isValid() ? "YES" : "NO") + "</li>";
  html += "<li>satellites: " + String(gps.satellites.isValid() ? gps.satellites.value() : 0) + "</li>";
  html += "<li>hdop: " + String(gps.hdop.isValid() ? gps.hdop.hdop() : 0.0) + "</li>";
  html += "</ul>";

  html += "<h3>Last NMEA (raw)</h3><pre>";
  html += lastNmeaLine;
  html += "</pre>";

  html += "<h3>Last RMC</h3><pre>";
  html += lastRmcLine;
  html += "</pre>";

  html += "<h3>Last GGA</h3><pre>";
  html += lastGgaLine;
  html += "</pre>";

  html += "</body></html>";

  web.send(200, "text/html; charset=utf-8", html);
}

// -------------------- NTP --------------------
void handleNtpRequest() {
  int packetSize = udp.parsePacket();
  if (packetSize < 48) {
    if (packetSize > 0) udp.flush();
    return;
  }

  uint8_t req[48];
  udp.read(req, 48);

  // NOTE: we still only answer when we have a valid GPS-derived time.
  // If you want, we can change this to reply with "unsynchronized" (Stratum 16).
  if (!haveTime) return;

  uint32_t elapsedMs = millis() - lastSyncMillis;
  uint32_t nowUnix   = lastUnixTime + (elapsedMs / 1000UL);
  uint32_t fracMs    = elapsedMs % 1000UL;

  uint32_t nowNtpSec  = nowUnix + NTP_EPOCH_OFFSET;
  uint32_t nowNtpFrac = (uint32_t)((double)fracMs / 1000.0 * 4294967296.0);

  uint8_t resp[48] = {0};

  // LI=0, VN=4, Mode=4 (server)
  resp[0] = 0b00100100;
  resp[1] = 2;   // Stratum: 2 (honest-ish without PPS discipline)
  resp[2] = 6;   // Poll
  resp[3] = (uint8_t)(-18); // Precision (rough)

  // Reference ID "GPS"
  resp[12] = 'G'; resp[13] = 'P'; resp[14] = 'S'; resp[15] = 0;

  // Originate Timestamp = client's Transmit Timestamp
  memcpy(&resp[24], &req[40], 8);

  // Reference Timestamp = last synced second
  uint32_t refNtpSec = lastUnixTime + NTP_EPOCH_OFFSET;
  resp[16] = (refNtpSec >> 24) & 0xFF;
  resp[17] = (refNtpSec >> 16) & 0xFF;
  resp[18] = (refNtpSec >>  8) & 0xFF;
  resp[19] = (refNtpSec >>  0) & 0xFF;

  // Receive Timestamp ~ now
  resp[32] = (nowNtpSec >> 24) & 0xFF;
  resp[33] = (nowNtpSec >> 16) & 0xFF;
  resp[34] = (nowNtpSec >>  8) & 0xFF;
  resp[35] = (nowNtpSec >>  0) & 0xFF;
  resp[36] = (nowNtpFrac >> 24) & 0xFF;
  resp[37] = (nowNtpFrac >> 16) & 0xFF;
  resp[38] = (nowNtpFrac >>  8) & 0xFF;
  resp[39] = (nowNtpFrac >>  0) & 0xFF;

  // Transmit Timestamp ~ now
  resp[40] = (nowNtpSec >> 24) & 0xFF;
  resp[41] = (nowNtpSec >> 16) & 0xFF;
  resp[42] = (nowNtpSec >>  8) & 0xFF;
  resp[43] = (nowNtpSec >>  0) & 0xFF;
  resp[44] = (nowNtpFrac >> 24) & 0xFF;
  resp[45] = (nowNtpFrac >> 16) & 0xFF;
  resp[46] = (nowNtpFrac >>  8) & 0xFF;
  resp[47] = (nowNtpFrac >>  0) & 0xFF;

  udp.beginPacket(udp.remoteIP(), udp.remotePort());
  udp.write(resp, 48);
  udp.endPacket();
}

void setup() {
  Serial.begin(115200);
  delay(200);

  gpsSerial.begin(GPS_BAUD);

  // Give GPS some time after power-up
  delay(600);
  configureGPS_RMC_GGA_only();
  // Optional: save into GPS non-volatile memory (may or may not persist depending on module)
  // saveGPSConfig();

  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  Serial.print("Connecting WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  // NTP
  udp.begin(NTP_PORT);
  Serial.println("NTP server started on UDP/123");

  // HTTP status page
  web.on("/", handleRoot);
  web.begin();
  Serial.print("HTTP status page: http://");
  Serial.print(WiFi.localIP());
  Serial.println("/");
}

void loop() {
  // Parse GPS + capture NMEA lines for remote debug
  while (gpsSerial.available() > 0) {
    char c = (char)gpsSerial.read();
    gps.encode(c);

    if (c == '\n') {
      lastNmeaLine.trim();

      if (lastNmeaLine.startsWith("$GPRMC") || lastNmeaLine.startsWith("$GNRMC")) lastRmcLine = lastNmeaLine;
      if (lastNmeaLine.startsWith("$GPGGA") || lastNmeaLine.startsWith("$GNGGA")) lastGgaLine = lastNmeaLine;

      lastNmeaLine = "";
    } else if (c != '\r') {
      if (lastNmeaLine.length() < 220) lastNmeaLine += c;
    }
  }

  // Sync base time when we get a fresh RMC update
  if (gps.date.isValid() && gps.time.isValid() && gps.date.isUpdated() && gps.time.isUpdated()) {
    int year  = gps.date.year();
    int month = gps.date.month();
    int day   = gps.date.day();
    int hour  = gps.time.hour();
    int min   = gps.time.minute();
    int sec   = gps.time.second();

    if (year >= 2020 && month >= 1 && month <= 12 && day >= 1 && day <= 31) {
      lastUnixTime = toUnix(year, month, day, hour, min, sec);
      lastSyncMillis = millis();
      haveTime = true;
    }
  }

  handleNtpRequest();
  web.handleClient();

  // Status every 2s (optional; useful when USB is connected)
  static uint32_t lastPrint = 0;
  if (millis() - lastPrint > 2000) {
    lastPrint = millis();

    Serial.print("haveTime=");
    Serial.print(haveTime ? "YES" : "NO");
    Serial.print(" timeValid=");
    Serial.print(gps.time.isValid() ? "YES" : "NO");
    Serial.print(" dateValid=");
    Serial.print(gps.date.isValid() ? "YES" : "NO");

    Serial.print(" fix=");
    Serial.print(gps.location.isValid() ? "YES" : "NO");
    Serial.print(" sats=");
    Serial.print(gps.satellites.isValid() ? gps.satellites.value() : 0);

    Serial.print(" hdop=");
    Serial.println(gps.hdop.isValid() ? gps.hdop.hdop() : 0.0);
  }
}
