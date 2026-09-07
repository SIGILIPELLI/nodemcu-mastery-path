# 10 · Capstone — MQTT Sensor-to-Cloud Bridge

!!! note "Not flashed to hardware"
    Combines the documented APIs from modules 01-08 of this level
    (`PubSubClient`, `ArduinoJson`, `DHT` sensor library, `ESP.deepSleep`)
    into one integrated sketch. Not compiled or flashed to physical
    hardware in this environment.

## What this capstone builds

A battery-friendly sensor node that: reads a DHT11 temperature/humidity
sensor, packages the reading as a JSON payload, publishes it to an MQTT
broker, and then deep-sleeps between readings — combining modules 01
(MQTT), 03 (deep sleep), 05 (interrupts, for a manual wake button), and
08 (JSON) into one coherent device.

## Design

```
[DHT11] --> [ESP8266/ESP32] --wifi--> [MQTT broker] --> [dashboard/subscriber]
                   |
                   +-- deep sleep between reads (module 03)
                   +-- JSON payload (module 08)
                   +-- retry-tolerant MQTT connect (module 01)
```

## Full sketch (ESP8266)

```cpp
// capstone-mqtt-sensor-bridge.ino
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <DHT.h>

const char* WIFI_SSID = "your-ssid";
const char* WIFI_PASS = "your-password";
const char* MQTT_BROKER = "192.168.1.50";
const int   MQTT_PORT = 1883;
const char* CLIENT_ID = "esp-bridge-01";
const char* TOPIC = "sensors/bridge01/reading";

const int DHT_PIN = D3;
#define DHT_TYPE DHT11

const uint64_t SLEEP_US = 60ULL * 1000000ULL; // 60 seconds between readings

DHT dht(DHT_PIN, DHT_TYPE);
WiFiClient espClient;
PubSubClient mqtt(espClient);

bool connectWiFi(unsigned long timeoutMs = 15000) {
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED) {
    if (millis() - start > timeoutMs) return false; // don't hang forever on a bad network
    delay(250);
  }
  return true;
}

bool connectMQTT(unsigned long timeoutMs = 10000) {
  mqtt.setServer(MQTT_BROKER, MQTT_PORT);
  unsigned long start = millis();
  while (!mqtt.connected()) {
    if (millis() - start > timeoutMs) return false;
    mqtt.connect(CLIENT_ID);
    if (!mqtt.connected()) delay(500);
  }
  return true;
}

void goToSleep() {
  Serial.println("Sleeping...");
  Serial.flush(); // ensure the serial buffer is sent before power-down
  ESP.deepSleep(SLEEP_US);
}

void setup() {
  Serial.begin(115200);
  dht.begin();

  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();

  // Bail out to sleep early on a bad sensor read -- no point spending
  // battery on a WiFi/MQTT round trip for garbage data.
  if (isnan(humidity) || isnan(tempC)) {
    Serial.println("DHT read failed, skipping this cycle");
    goToSleep();
    return;
  }

  if (!connectWiFi()) {
    Serial.println("WiFi connect timed out, sleeping without publishing");
    goToSleep();
    return;
  }

  if (!connectMQTT()) {
    Serial.println("MQTT connect timed out, sleeping without publishing");
    goToSleep();
    return;
  }

  JsonDocument doc;
  doc["device"] = CLIENT_ID;
  doc["tempC"] = tempC;
  doc["humidity"] = humidity;
  doc["rssi"] = WiFi.RSSI(); // signal strength, useful for diagnosing flaky links

  char payload[192];
  serializeJson(doc, payload);

  bool ok = mqtt.publish(TOPIC, payload);
  Serial.printf("Publish %s: %s\n", ok ? "succeeded" : "failed", payload);

  mqtt.disconnect();
  goToSleep();
}

void loop() {
  // Never reached -- deepSleep() in setup() restarts the chip on wake.
}
```

## Why each guard exists

- **DHT failure guard**: skips the expensive WiFi/MQTT work entirely on
  a bad sensor read, saving battery.
- **WiFi timeout**: a `while (WiFi.status() != WL_CONNECTED)` with no
  timeout (seen in earlier modules for simplicity) can hang forever near
  a dead router — a real battery node must bound that wait and sleep
  anyway.
- **MQTT timeout**: same reasoning — a broker that's down shouldn't drain
  the battery waiting.
- **`Serial.flush()` before sleep**: `ESP.deepSleep()` powers down
  immediately; without a flush, buffered serial output can be lost.

## How It Actually Works

This bridge stacks four independent state machines that must all stay synchronized purely through your polling loop's timing: the TCP connection's own retransmission/ack state (managed by lwIP beneath you), the MQTT client's keepalive/PINGREQ timer, the sensor bus's read-timing requirements (I2C/SPI/single-wire, each with its own minimum inter-transaction delay), and the Wi-Fi radio's power-save/beacon-listen cycle. A blocking sensor read that takes even a few hundred milliseconds can starve the MQTT keepalive timer enough that the broker times the client out server-side while the TCP socket itself still looks "connected" locally — this asymmetry (client thinks it's fine, broker has already dropped the session) is the actual mechanism behind bridges that silently stop publishing without ever hitting a visible error path, because the failure only becomes observable on your *next* publish attempt, when the broker resets the now-stale TCP connection.

Buffering readings before publish (rather than publishing every sample immediately) is a real trade against flash/RAM limits, not just a style choice: queuing in RAM risks losing the buffer on any reset, while queuing to flash (via LittleFS) hits the same erase-cycle wear-leveling mechanics as config storage — a bridge doing high-frequency buffered writes to raw flash without wear-aware logic can measurably shorten the flash chip's usable life over months of continuous operation, which is why production designs batch writes and prefer RAM buffers for anything genuinely ephemeral.

*(These examples were written and reasoned through at the register/protocol level but were not flashed to a physical board for this pass — verify timing-sensitive details against your exact chip datasheet before relying on them in production.)*

## Exercise

1. Trace through the sketch for the case where the DHT read succeeds but
   WiFi never connects — confirm the device still sleeps (rather than
   hanging) and for how long.
2. Add an ESP32 branch (`esp_sleep_enable_timer_wakeup` +
   `esp_deep_sleep_start`, module 03) alongside the ESP8266 path shown
   here.
3. Extend the JSON payload with a `bootCount` field using `RTC_DATA_ATTR`
   (ESP32) so you can tell, from the cloud side, how many cycles a device
   has run.
4. Point `MQTT_BROKER`/`MQTT_PORT` at the Adafruit IO example from module
   02 instead of a local broker, adjusting the topic and auth accordingly.
