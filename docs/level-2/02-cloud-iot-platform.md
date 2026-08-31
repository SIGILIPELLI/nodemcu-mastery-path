# 02 · Connecting to a Cloud IoT Platform

!!! note "Not flashed to hardware"
    Reasoned through against Adafruit IO's publicly documented MQTT API
    (host `io.adafruit.com`, port `8883` TLS / `1883` plain, feed-based
    topic naming `<username>/feeds/<feedname>`) and the `PubSubClient`
    library's documented API. Not compiled or flashed to physical
    hardware in this environment.

## From a local broker to the internet

Level 2's first module used a broker on your LAN. Real IoT projects
usually publish to a **cloud IoT platform** instead, so data is reachable
from anywhere and you get dashboards, storage, and alerting for free.
**Adafruit IO** is a good learning platform: free tier, plain MQTT
support, and a simple feed model — every value you publish belongs to a
named "feed," and each feed gets an auto-generated web dashboard.

## Getting credentials

1. Create a free account at Adafruit IO and note your **username** and
   **AIO Key** (Adafruit IO's API key, found under "My Key").
2. Create a feed named `temperature` in the dashboard — Adafruit IO auto-
   creates feeds on first publish too, but creating one explicitly first
   makes topic naming obvious.

## Publishing to Adafruit IO over MQTT

```cpp
// cloud-publish-adafruitio.ino
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

const char* WIFI_SSID = "your-ssid";
const char* WIFI_PASS = "your-password";

const char* AIO_SERVER = "io.adafruit.com";
const int   AIO_PORT = 1883; // plain MQTT; use 8883 + WiFiClientSecure for TLS
const char* AIO_USERNAME = "your-aio-username";
const char* AIO_KEY = "your-aio-key";

// Adafruit IO's documented topic convention: <username>/feeds/<feedname>
String feedTopic = String(AIO_USERNAME) + "/feeds/temperature";

WiFiClient espClient;
PubSubClient mqtt(espClient);

void connectWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);
}

void connectMQTT() {
  mqtt.setServer(AIO_SERVER, AIO_PORT);
  while (!mqtt.connected()) {
    // Adafruit IO authenticates via MQTT username/password fields, not
    // a separate handshake -- username is your AIO username, password
    // is your AIO key.
    if (mqtt.connect("esp-node-01", AIO_USERNAME, AIO_KEY)) {
      Serial.println("Adafruit IO connected");
    } else {
      Serial.printf("failed, rc=%d\n", mqtt.state());
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  connectWiFi();
  connectMQTT();
}

void loop() {
  if (!mqtt.connected()) connectMQTT();
  mqtt.loop();

  static unsigned long last = 0;
  if (millis() - last > 15000) { // Adafruit IO's free tier rate-limits publishes
    last = millis();
    float tempC = 23.4;
    char payload[16];
    dtostrf(tempC, 4, 1, payload);
    mqtt.publish(feedTopic.c_str(), payload);
    Serial.printf("Published %s to %s\n", payload, feedTopic.c_str());
  }
}
```

The 15-second publish interval isn't arbitrary — Adafruit IO's free tier
documents a rate limit (roughly 30 data points/minute), and publishing
faster gets throttled or dropped.

## Subscribing to a feed for remote control

The same feed topic works both ways — publish sensor data on one feed,
subscribe to a different feed (e.g. `led`) so the dashboard's toggle
switch can control the device:

```cpp
// cloud-subscribe-adafruitio.ino
void onMessage(char* topic, byte* payload, unsigned int length) {
  String message;
  for (unsigned int i = 0; i < length; i++) message += (char)payload[i];
  Serial.printf("[%s] %s\n", topic, message.c_str());
  digitalWrite(LED_BUILTIN, message == "ON" ? LOW : HIGH);
}

// In setup(), after connectMQTT():
//   mqtt.setCallback(onMessage);
//   mqtt.subscribe((String(AIO_USERNAME) + "/feeds/led").c_str());
```

## Exercise

1. Sign up for Adafruit IO, create a `temperature` feed, and confirm the
   publish sketch's values appear on its dashboard chart.
2. Add a toggle-switch block to an `led` feed in the dashboard and wire
   up the subscribe sketch so flipping it (conceptually) drives an LED.
3. Deliberately use a wrong AIO key and confirm `mqtt.state()` reports a
   non-zero code consistent with an authorization failure (Adafruit IO
   and PubSubClient both document this behavior).
4. Note where this design is insecure (`AIO_PORT 1883`, plain-text key
   over the network) — Level 3's TLS module addresses it directly.
