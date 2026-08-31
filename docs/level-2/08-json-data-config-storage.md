# 08 · JSON Data & Config Storage

!!! note "Not flashed to hardware"
    Reasoned through against the widely-used `ArduinoJson` library's
    (by Benoit Blanchon) documented v6/v7 API
    (`JsonDocument`, `serializeJson()`, `deserializeJson()`) and the
    ESP8266/ESP32 Arduino cores' documented `LittleFS` filesystem API.
    Not compiled or flashed to physical hardware in this environment.

## Why JSON on a microcontroller

Cloud IoT platforms, REST APIs (Level 3), and config files all tend to
speak **JSON**. Rather than hand-parsing strings, the standard library is
**ArduinoJson** — memory-efficient, well documented, and designed
specifically for constrained devices like the ESP8266/ESP32.

Install it via **Sketch → Include Library → Manage Libraries…**, search
`ArduinoJson`, install the latest v6 or v7 release (this module uses the
v6/v7-compatible `JsonDocument` API, ArduinoJson's current documented
interface).

## Serializing: building a JSON payload to publish

```cpp
// json-serialize-basic.ino
#include <ArduinoJson.h>

void setup() {
  Serial.begin(115200);

  // JsonDocument auto-sizes its internal buffer (ArduinoJson v7's
  // documented replacement for the older fixed-capacity
  // StaticJsonDocument/DynamicJsonDocument types).
  JsonDocument doc;
  doc["device"] = "esp-livingroom-01";
  doc["tempC"] = 22.5;
  doc["humidity"] = 48;
  doc["online"] = true;

  char output[256];
  size_t len = serializeJson(doc, output); // writes JSON text into output
  Serial.printf("Payload (%d bytes): %s\n", len, output);
  // {"device":"esp-livingroom-01","tempC":22.5,"humidity":48,"online":true}
}

void loop() {}
```

That `output` buffer is exactly what you'd hand to `mqtt.publish()`
(module 01) or an HTTP client body — JSON as text, ready to send.

## Deserializing: parsing an incoming JSON message

```cpp
// json-deserialize-basic.ino
#include <ArduinoJson.h>

void handleIncoming(const char* json) {
  JsonDocument doc;
  // deserializeJson() returns a DeserializationError -- documented as
  // falsy (== ok) on success, truthy on any parse failure.
  DeserializationError err = deserializeJson(doc, json);
  if (err) {
    Serial.printf("JSON parse failed: %s\n", err.c_str());
    return;
  }

  const char* device = doc["device"];
  float tempC = doc["tempC"];
  bool online = doc["online"];

  Serial.printf("Device: %s  Temp: %.1f  Online: %d\n", device, tempC, online);
}

void setup() {
  Serial.begin(115200);
  handleIncoming("{\"device\":\"esp-livingroom-01\",\"tempC\":22.5,\"online\":true}");
  handleIncoming("{not valid json"); // demonstrates the error path
}

void loop() {}
```

## Nested objects and arrays

```cpp
// json-nested-example.ino
JsonDocument doc;
doc["device"] = "esp-livingroom-01";

JsonObject location = doc["location"].to<JsonObject>();
location["lat"] = 17.385;
location["lon"] = 78.4867;

JsonArray readings = doc["readings"].to<JsonArray>();
readings.add(22.1);
readings.add(22.4);
readings.add(22.6);

char output[256];
serializeJson(doc, output);
// {"device":"esp-livingroom-01","location":{"lat":17.385,"lon":78.4867},
//  "readings":[22.1,22.4,22.6]}
```

## Persisting config to flash with LittleFS

Rather than hardcoding WiFi credentials or thresholds in the sketch,
saving them as a JSON file on the filesystem lets you change config
without reflashing:

```cpp
// json-config-littlefs.ino
#include <LittleFS.h>
#include <ArduinoJson.h>

struct Config {
  String wifiSsid;
  int publishIntervalMs;
};

bool loadConfig(Config &cfg) {
  if (!LittleFS.begin()) {
    Serial.println("LittleFS mount failed");
    return false;
  }
  // LittleFS.exists()/open() document standard POSIX-like file semantics.
  if (!LittleFS.exists("/config.json")) {
    Serial.println("No config.json found, using defaults");
    return false;
  }
  File f = LittleFS.open("/config.json", "r");
  JsonDocument doc;
  DeserializationError err = deserializeJson(doc, f);
  f.close();
  if (err) {
    Serial.println("Config JSON invalid");
    return false;
  }
  cfg.wifiSsid = doc["wifiSsid"].as<String>();
  cfg.publishIntervalMs = doc["publishIntervalMs"] | 5000; // default if missing
  return true;
}

void saveConfig(const Config &cfg) {
  JsonDocument doc;
  doc["wifiSsid"] = cfg.wifiSsid;
  doc["publishIntervalMs"] = cfg.publishIntervalMs;

  File f = LittleFS.open("/config.json", "w");
  serializeJson(doc, f); // ArduinoJson can serialize directly to a Stream
  f.close();
}

void setup() {
  Serial.begin(115200);
  Config cfg;
  if (!loadConfig(cfg)) {
    cfg.wifiSsid = "default-ssid";
    cfg.publishIntervalMs = 5000;
    saveConfig(cfg);
  }
  Serial.printf("SSID: %s  Interval: %dms\n",
                cfg.wifiSsid.c_str(), cfg.publishIntervalMs);
}

void loop() {}
```

The `doc["publishIntervalMs"] | 5000` syntax is ArduinoJson's documented
"get with default" idiom — it returns the stored value if the key exists
and is convertible, otherwise the fallback, which is exactly what you
want for a config field added in a later firmware version.

## Exercise

1. Write the serialize sketch and confirm (by tracing the code) the
   exact JSON string it would produce.
2. Write the deserialize sketch, then feed it a payload with a missing
   field and reason through what value that field would take.
3. Build the nested example and add a second array element containing an
   object (e.g. `{"sensor":"bme280","status":"ok"}`).
4. Write the LittleFS config sketch, then simulate a "second boot" by
   reasoning through what `loadConfig()` returns when `/config.json`
   already exists from a previous `saveConfig()` call.
