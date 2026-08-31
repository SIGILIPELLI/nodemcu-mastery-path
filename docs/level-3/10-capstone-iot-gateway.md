# Capstone — Secure Multi-Sensor IoT Gateway

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP32 Arduino-core APIs
    used throughout this level (`WiFiClientSecure`, `PubSubClient`/MQTT,
    `LittleFS`, `esp_task_wdt`, `Ticker`). Not compiled or flashed to
    physical hardware in this environment.

## What this capstone combines

Every prior Level 3 module becomes one subsystem of a single gateway
sketch:

- **REST API (03-01)** → a local status endpoint for on-LAN debugging.
- **TLS (03-02)** → the uplink to the cloud MQTT broker is encrypted.
- **Multi-sensor aggregation (03-03)** → several sensors sampled and
  buffered before publish.
- **Watchdog (03-04)** → recovers from any subsystem hang.
- **Task scheduling (03-05)** → everything runs non-blocking on one
  loop.
- **Local logging (03-06)** → readings survive a network outage.
- **Captive portal (03-07)** → first-boot WiFi provisioning.
- **Firmware rollback (03-08)** → OTA updates that self-validate.
- **Diagnostics (03-09)** → health telemetry alongside sensor data.

## Architecture

```
[ DHT22 ]--\
[ BMP280 ]---> aggregate & buffer --> LittleFS (offline buffer)
[ Analog ]--/         |                      |
                       v                      v
                 MQTT publish (TLS) <---- drain on reconnect
                       |
                 local REST /status (LAN debugging)
```

## The combined sketch (structured, trimmed for length)

```cpp
// iot-gateway-capstone.ino
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <WebServer.h>
#include <LittleFS.h>
#include <esp_task_wdt.h>
#include <DNSServer.h>

// ---- Config ----
const char* MQTT_HOST = "broker.example.com";
const int   MQTT_PORT = 8883;
const char* MQTT_TOPIC = "gateway/readings";
const unsigned long SAMPLE_INTERVAL_MS  = 5000;
const unsigned long PUBLISH_INTERVAL_MS = 30000;
const unsigned long WDT_TIMEOUT_S = 8;

WiFiClientSecure tlsClient;
PubSubClient mqtt(tlsClient);
WebServer statusServer(80);
DNSServer dnsServer;

unsigned long lastSample = 0, lastPublish = 0;
float latestTemp = NAN, latestHumidity = NAN, latestPressure = NAN;
bool provisioningMode = false;

// ---- Provisioning (Level 3.07) ----
bool tryStoredWifi() {
  File f = LittleFS.open("/wifi.cfg", "r");
  if (!f) return false;
  String ssid = f.readStringUntil('\n'); ssid.trim();
  String pass = f.readStringUntil('\n'); pass.trim();
  f.close();
  if (ssid.length() == 0) return false;

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid.c_str(), pass.c_str());
  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 15000) {
    delay(200);
    esp_task_wdt_reset();
  }
  return WiFi.status() == WL_CONNECTED;
}

void startProvisioningPortal() {
  provisioningMode = true;
  WiFi.mode(WIFI_AP);
  WiFi.softAP("Gateway-Setup");
  dnsServer.start(53, "*", WiFi.softAPIP());

  statusServer.on("/", HTTP_GET, []() {
    statusServer.send(200, "text/html",
      "<form method='POST' action='/save'>SSID:<input name='ssid'>"
      "Pass:<input name='pass' type='password'>"
      "<input type='submit'></form>");
  });
  statusServer.on("/save", HTTP_POST, []() {
    File f = LittleFS.open("/wifi.cfg", "w");
    if (f) {
      f.println(statusServer.arg("ssid"));
      f.println(statusServer.arg("pass"));
      f.close();
    }
    statusServer.send(200, "text/plain", "Saved. Rebooting.");
    delay(1000);
    ESP.restart();
  });
  statusServer.onNotFound([]() { statusServer.send(200, "text/html", "Setup"); });
  statusServer.begin();
}

// ---- Sensor aggregation (Level 3.03) ----
void sampleSensors() {
  // Stand-ins: real reads would use DHT.h / Adafruit_BMP280 as covered
  // in Level 1/3 sensor modules.
  latestTemp = 21.0 + (random(-20, 20) / 10.0);
  latestHumidity = 45.0 + (random(-50, 50) / 10.0);
  latestPressure = 1013.0 + (random(-30, 30) / 10.0);
}

// ---- Offline buffering (Level 3.06) ----
void bufferReadingLocally() {
  File f = LittleFS.open("/buffer.csv", "a");
  if (f) {
    f.printf("%lu,%.2f,%.2f,%.2f\n", millis(), latestTemp, latestHumidity, latestPressure);
    f.close();
  }
}

bool publishBufferedReadings() {
  File f = LittleFS.open("/buffer.csv", "r");
  if (!f) return true; // nothing buffered
  bool allSent = true;
  String remaining = "";
  while (f.available()) {
    String line = f.readStringUntil('\n');
    if (line.length() == 0) continue;
    if (mqtt.connected() && mqtt.publish(MQTT_TOPIC, line.c_str())) {
      // sent
    } else {
      remaining += line + "\n";
      allSent = false;
    }
  }
  f.close();
  LittleFS.remove("/buffer.csv");
  if (remaining.length() > 0) {
    File rf = LittleFS.open("/buffer.csv", "w");
    if (rf) { rf.print(remaining); rf.close(); }
  }
  return allSent;
}

// ---- MQTT over TLS (Level 3.02) ----
bool ensureMqttConnected() {
  if (mqtt.connected()) return true;
  tlsClient.setInsecure(); // lab shortcut; production pins setCACert()
  mqtt.setServer(MQTT_HOST, MQTT_PORT);
  return mqtt.connect("gateway-01");
}

// ---- Local REST status endpoint (Level 3.01) ----
void setupStatusApi() {
  statusServer.on("/status", HTTP_GET, []() {
    String json = "{";
    json += "\"uptime_ms\":" + String(millis()) + ",";
    json += "\"free_heap\":" + String(ESP.getFreeHeap()) + ",";
    json += "\"temp_c\":" + String(latestTemp, 1) + ",";
    json += "\"humidity\":" + String(latestHumidity, 1) + ",";
    json += "\"mqtt_connected\":" + String(mqtt.connected() ? "true" : "false");
    json += "}";
    statusServer.send(200, "application/json", json);
  });
  statusServer.begin();
}

void setup() {
  Serial.begin(115200);
  LittleFS.begin(true);
  esp_task_wdt_init(WDT_TIMEOUT_S, true);
  esp_task_wdt_add(NULL);

  if (!tryStoredWifi()) {
    startProvisioningPortal();
    return; // stay in provisioning mode; loop() below skips gateway logic
  }

  setupStatusApi();
  Serial.println("Gateway online: " + WiFi.localIP().toString());
}

void loop() {
  esp_task_wdt_reset();

  if (provisioningMode) {
    dnsServer.processNextRequest();
    statusServer.handleClient();
    return;
  }

  unsigned long now = millis();

  if (now - lastSample >= SAMPLE_INTERVAL_MS) {
    lastSample = now;
    sampleSensors();
    bufferReadingLocally();
  }

  if (now - lastPublish >= PUBLISH_INTERVAL_MS) {
    lastPublish = now;
    if (ensureMqttConnected()) {
      publishBufferedReadings();
    }
  }

  mqtt.loop();
  statusServer.handleClient();
}
```

## Why this holds up as a capstone

Every piece is non-blocking (task-scheduling module), survives a
network outage (local-logging module) without losing data
(watchdog-safe reconnect loop), can be provisioned without hardcoded
credentials (captive-portal module), and exposes both a local debug
surface (REST) and a remote one (MQTT + would-be diagnostics logger from
03-09) for observing it in the field.

## Exercise

1. Add the reset-reason and free-heap diagnostics from 03-09 into the
   MQTT publish payload, so cloud-side dashboards see device health
   alongside sensor data.
2. Wire in the firmware-rollback self-test pattern from 03-08: after an
   OTA update, require `ensureMqttConnected()` to succeed within 30s or
   trigger a rollback.
3. The `publishBufferedReadings()` function re-buffers only lines that
   failed to publish — trace through what happens if `mqtt.publish()`
   returns true but the broker never actually received the message
   (e.g. TCP reset mid-write). Is this design safe against that case?
4. Add a hardware "factory reset" trigger (button held during boot)
   that calls `LittleFS.remove("/wifi.cfg")` before the
   `tryStoredWifi()` check.
