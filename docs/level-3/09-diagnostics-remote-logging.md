# Diagnostics & Remote Logging

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP8266/ESP32 `ESP.*`
    diagnostic APIs and `WiFiClientSecure`/`HTTPClient` networking APIs.
    Not compiled or flashed to physical hardware in this environment.

## You can't `Serial.println()` a deployed device

Once a board is in the field, USB Serial is gone. Diagnosing "device
#47 stopped reporting" needs telemetry the device pushes itself:
uptime, free heap, WiFi signal, last reset reason, and structured log
lines — sent somewhere you can actually query them.

## Step 1: collect the built-in health metrics

Both cores document `ESP.*` calls that expose runtime health without
any extra library:

```cpp
// health-snapshot.ino
#if defined(ESP8266)
  #include <ESP8266WiFi.h>
#else
  #include <WiFi.h>
  #include <esp_system.h>
#endif

struct HealthSnapshot {
  unsigned long uptimeMs;
  uint32_t freeHeap;
  int8_t rssi;
  String resetReason;
};

HealthSnapshot collectHealth() {
  HealthSnapshot h;
  h.uptimeMs = millis();
  h.freeHeap = ESP.getFreeHeap(); // documented on both cores
  h.rssi = WiFi.RSSI();          // documented on both cores, dBm

#if defined(ESP8266)
  h.resetReason = ESP.getResetReason(); // e.g. "Power on", "Watchdog"
#else
  esp_reset_reason_t r = esp_reset_reason();
  h.resetReason = (r == ESP_RST_TASK_WDT || r == ESP_RST_WDT)
                    ? "Watchdog" : "Other";
#endif
  return h;
}
```

## Step 2: a leveled logger that buffers, then ships

```cpp
// remote-logger.h
#pragma once
#include <Arduino.h>

enum LogLevel { LOG_DEBUG, LOG_INFO, LOG_WARN, LOG_ERROR };

class RemoteLogger {
public:
  void log(LogLevel level, const String& msg) {
    String line = levelName(level) + " " + String(millis()) + " " + msg;
    Serial.println(line);       // still useful on the bench
    buffer += line + "\n";
    if (buffer.length() > 2048) flush(); // avoid unbounded RAM growth
  }

  void flush() {
    if (buffer.length() == 0) return;
    if (sendToServer(buffer)) {
      buffer = "";
    }
    // On failure, keep buffering (bounded by the 2048-byte cap above)
    // and retry on the next flush() call.
  }

private:
  String buffer;

  String levelName(LogLevel l) {
    switch (l) {
      case LOG_DEBUG: return "DEBUG";
      case LOG_INFO:  return "INFO";
      case LOG_WARN:  return "WARN";
      default:        return "ERROR";
    }
  }

  bool sendToServer(const String& payload); // implemented per-transport below
};
```

## Step 3: shipping logs over HTTPS

```cpp
// http-log-shipper.ino
#if defined(ESP8266)
  #include <WiFiClientSecure.h>
  #include <ESP8266HTTPClient.h>
#else
  #include <WiFiClientSecure.h>
  #include <HTTPClient.h>
#endif

bool RemoteLogger::sendToServer(const String& payload) {
  WiFiClientSecure client;
  client.setInsecure(); // documented shortcut for lab use; a production
                         // build should pin a CA cert via setCACert()

  HTTPClient http;
  if (!http.begin(client, "https://logs.example.com/ingest")) {
    return false;
  }
  http.addHeader("Content-Type", "text/plain");
  int code = http.POST(payload);
  http.end();

  return code == 200;
}
```

`HTTPClient::begin(WiFiClientSecure&, url)` and `.POST()` are documented
on both the ESP8266 and ESP32 HTTPClient libraries with the same
signature, which is why only the `#include` differs between cores.

## Step 4: periodic health pings alongside logs

```cpp
// diagnostics-loop.ino
RemoteLogger logger;
unsigned long lastHealthPing = 0;
const unsigned long HEALTH_INTERVAL_MS = 60000;

void loop() {
  unsigned long now = millis();
  if (now - lastHealthPing >= HEALTH_INTERVAL_MS) {
    lastHealthPing = now;
    HealthSnapshot h = collectHealth();
    logger.log(LOG_INFO,
      "heap=" + String(h.freeHeap) +
      " rssi=" + String(h.rssi) +
      " uptime=" + String(h.uptimeMs) +
      " reset=" + h.resetReason);
    logger.flush();
  }
}
```

## Watching for the metrics that predict failure

A single free-heap reading means little; a *downward trend* across many
pings means a memory leak headed for a crash. Two cheap early-warning
checks:

```cpp
uint32_t minHeapSeen = UINT32_MAX;

void checkHeapTrend() {
  uint32_t heap = ESP.getFreeHeap();
  if (heap < minHeapSeen) minHeapSeen = heap;

  if (heap < 4096) { // documented low-memory territory on both cores
    logger.log(LOG_WARN, "Low heap: " + String(heap) + " bytes free");
  }
}
```

Correlating this alongside the reset-reason logging from the watchdog
module turns "device #47 went silent" into "device #47's heap trended
down for six hours, then watchdog-reset" — an actionable root cause
instead of a mystery.

## Exercise

1. Implement `sendToServer()` using MQTT (from Level 2) instead of
   HTTPS, publishing to a topic like `devices/<id>/logs`.
2. Add a `LOG_ERROR` call inside a sensor-read failure path and confirm
   (by tracing the code) that `flush()` is triggered promptly rather
   than waiting for the buffer to fill.
3. Extend `checkHeapTrend()` to log a warning only when heap has
   decreased for three consecutive checks in a row (a real leak
   signature, versus normal fluctuation).
4. Explain the security tradeoff of `client.setInsecure()` and describe
   the documented alternative (`setCACert()`) for a production
   deployment.
