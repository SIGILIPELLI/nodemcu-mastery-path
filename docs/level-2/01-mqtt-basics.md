# 01 · MQTT Basics for IoT Messaging

!!! note "Not flashed to hardware"
    Reasoned through against the documented public API of the
    `PubSubClient` library by Nick O'Leary (the de facto standard
    lightweight MQTT client for Arduino on ESP8266/ESP32) and the
    publicly documented MQTT 3.1.1 protocol semantics (broker, topic,
    QoS, retained messages). Not compiled or flashed to physical
    hardware in this environment.

## Why MQTT instead of plain HTTP requests

Level 1 sketches used a web server the device hosted itself, or plain
HTTP requests the device made outward. That works for one device talking
to one client, but it doesn't scale to many devices reporting to many
listeners. **MQTT** ("Message Queuing Telemetry Transport") is a
publish/subscribe protocol built for exactly that: a lightweight
always-connected client (your ESP8266/ESP32) talks to a central
**broker**, publishing data to named **topics** ("home/livingroom/temp")
and subscribing to topics it cares about — the broker fans messages out
to every subscriber, and publishers never need to know who's listening.

Compared to HTTP polling, MQTT keeps one persistent TCP connection open
(no repeated connection setup/teardown), uses a much smaller message
framing overhead, and lets the broker push data to you the instant it's
published rather than you having to ask repeatedly.

## Installing PubSubClient and running a local broker

1. **Sketch → Include Library → Manage Libraries…**, search
   `PubSubClient` (by Nick O'Leary), install it.
2. For local testing you need an MQTT broker. **Mosquitto** is the
   standard open-source choice — install it on a computer on your LAN
   (`brew install mosquitto` on macOS, or run it in Docker) and it
   listens on port `1883` by default with no authentication out of the
   box, which is fine for learning but never for production (see
   Level 3's TLS module).

## Connecting and publishing

```cpp
// mqtt-publish-basic.ino
#include <ESP8266WiFi.h>   // swap for <WiFi.h> on ESP32
#include <PubSubClient.h>

const char* WIFI_SSID = "your-ssid";
const char* WIFI_PASS = "your-password";
const char* MQTT_BROKER = "192.168.1.50"; // your broker's LAN IP
const int   MQTT_PORT = 1883;
const char* CLIENT_ID = "esp-livingroom-01"; // must be unique per device

WiFiClient espClient;
PubSubClient mqtt(espClient);

void connectWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected: " + WiFi.localIP().toString());
}

void connectMQTT() {
  mqtt.setServer(MQTT_BROKER, MQTT_PORT);
  while (!mqtt.connected()) {
    Serial.print("Connecting to MQTT broker...");
    // PubSubClient::connect() returns true on success; the clientId
    // argument must be unique per device or the broker will disconnect
    // the older client with the same ID.
    if (mqtt.connect(CLIENT_ID)) {
      Serial.println("connected");
    } else {
      // state() returns a documented error code (e.g. -2 = connect failed,
      // -4 = connect timeout, 5 = not authorized)
      Serial.printf("failed, rc=%d, retrying in 5s\n", mqtt.state());
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
  if (!mqtt.connected()) {
    connectMQTT();
  }
  mqtt.loop(); // must be called regularly to process incoming/keepalive traffic

  static unsigned long lastPublish = 0;
  if (millis() - lastPublish > 5000) {
    lastPublish = millis();
    float tempC = 22.5; // stand-in for a real sensor reading
    char payload[16];
    dtostrf(tempC, 4, 1, payload); // PubSubClient::publish takes char*, not float
    mqtt.publish("home/livingroom/temp", payload);
    Serial.printf("Published: %s\n", payload);
  }
}
```

`PubSubClient::publish(topic, payload)` takes a topic string and a
payload as `const char*` or `uint8_t*` — there's no float overload, so
`dtostrf()` (a documented AVR/Arduino-core helper that formats a float
into a char buffer) converts the number to text first.

## Subscribing and handling incoming messages

Subscribing needs a callback function registered *before* `connect()`,
since the broker can deliver a message on a subscribed topic at any time
inside `mqtt.loop()`:

```cpp
// mqtt-subscribe-basic.ino
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

const char* MQTT_BROKER = "192.168.1.50";
WiFiClient espClient;
PubSubClient mqtt(espClient);

// The callback receives the topic and payload as a raw byte buffer --
// PubSubClient does NOT null-terminate it, so build a String carefully.
void onMessage(char* topic, byte* payload, unsigned int length) {
  String message;
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.printf("[%s] %s\n", topic, message.c_str());

  if (String(topic) == "home/livingroom/led" && message == "ON") {
    digitalWrite(LED_BUILTIN, LOW); // many boards wire the onboard LED active-low
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);
  // ... WiFi connect omitted for brevity, same as the publish example ...

  mqtt.setServer(MQTT_BROKER, 1883);
  mqtt.setCallback(onMessage); // must be set before connect()
  mqtt.connect("esp-livingroom-01");
  mqtt.subscribe("home/livingroom/led");
}

void loop() {
  mqtt.loop();
}
```

## Exercise

1. Install Mosquitto locally and confirm it's running with
   `mosquitto_sub -h localhost -t 'home/#'` in a terminal (`#` is MQTT's
   documented multi-level wildcard).
2. Flash (conceptually) the publish sketch and confirm the temp reading
   arrives in that same terminal.
3. Add the subscribe sketch and, from another terminal, run
   `mosquitto_pub -h localhost -t 'home/livingroom/led' -m 'ON'` — confirm
   the callback fires.
4. Change `CLIENT_ID` to the same value on two "devices" (two sketches
   pointed at the same broker) and observe the broker disconnect the
   first one when the second connects — this is why unique client IDs
   matter.
