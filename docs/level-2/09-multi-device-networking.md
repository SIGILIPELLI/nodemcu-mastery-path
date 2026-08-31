# 09 · Multi-Device Networking Patterns

!!! note "Not flashed to hardware"
    Reasoned through against the ESP8266/ESP32 Arduino cores' documented
    `WiFiUdp` (broadcast/multicast UDP) and ESP-NOW (`esp_now.h`,
    ESP32/ESP8266-specific low-level peer-to-peer protocol) APIs. Not
    compiled or flashed to physical hardware in this environment.

## Beyond one device talking to one broker

Modules 01-08 mostly assumed a single device talking to a broker or
cloud service. Real deployments often need devices to talk to **each
other** directly — a sensor network, a set of nodes electing a leader, or
a group of devices reacting to one node's event without round-tripping
through the internet. This module covers two complementary patterns:
**UDP broadcast** (works over regular WiFi, no router config needed
beyond normal association) and **ESP-NOW** (a lower-level, router-free
protocol built into both chips).

## UDP broadcast: one-to-many on the local network

UDP broadcast sends one packet that every device on the same subnet can
receive, without needing to know peer IP addresses in advance:

```cpp
// udp-broadcast-sender.ino
#include <ESP8266WiFi.h>
#include <WiFiUdp.h>

WiFiUDP udp;
const int UDP_PORT = 4210;

void setup() {
  Serial.begin(115200);
  WiFi.begin("your-ssid", "your-password");
  while (WiFi.status() != WL_CONNECTED) delay(500);
}

void loop() {
  // 255.255.255.255 is the documented limited-broadcast address --
  // delivered to every host on the local subnet, no routing needed.
  udp.beginPacket("255.255.255.255", UDP_PORT);
  udp.print("hello from esp-node-01");
  udp.endPacket();
  Serial.println("Broadcast sent");
  delay(2000);
}
```

```cpp
// udp-broadcast-receiver.ino
#include <ESP8266WiFi.h>
#include <WiFiUdp.h>

WiFiUDP udp;
const int UDP_PORT = 4210;
char packetBuffer[128];

void setup() {
  Serial.begin(115200);
  WiFi.begin("your-ssid", "your-password");
  while (WiFi.status() != WL_CONNECTED) delay(500);
  udp.begin(UDP_PORT); // start listening on this port
}

void loop() {
  // parsePacket() returns the packet size, or 0 if nothing's waiting --
  // documented as the non-blocking way to poll for UDP traffic.
  int packetSize = udp.parsePacket();
  if (packetSize) {
    int len = udp.read(packetBuffer, sizeof(packetBuffer) - 1);
    packetBuffer[len] = '\0';
    Serial.printf("Received from %s: %s\n",
                  udp.remoteIP().toString().c_str(), packetBuffer);
  }
}
```

UDP has no delivery guarantee (a documented property of the protocol
itself, not a library limitation) — fine for periodic heartbeats or
non-critical announcements, wrong for anything that must arrive exactly
once.

## ESP-NOW: router-free peer-to-peer

**ESP-NOW** is a connectionless protocol built into the ESP8266/ESP32
WiFi radio itself — no WiFi network join required, low latency, and
documented to work even while the chip is also connected to a regular
WiFi network (on ESP32). It needs each peer's **MAC address** registered
in advance.

```cpp
// esp-now-sender-esp32.ino
#include <WiFi.h>
#include <esp_now.h>

// Replace with the receiving device's actual MAC address.
uint8_t peerAddress[] = {0xAA, 0xBB, 0xCC, 0x11, 0x22, 0x33};

typedef struct {
  float tempC;
  int deviceId;
} SensorMessage;

void onDataSent(const uint8_t *mac, esp_now_send_status_t status) {
  Serial.println(status == ESP_NOW_SEND_SUCCESS ? "Send OK" : "Send failed");
}

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA); // ESP-NOW documented to require STA mode, even unconnected

  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW init failed");
    return;
  }
  esp_now_register_send_cb(onDataSent);

  esp_now_peer_info_t peerInfo = {};
  memcpy(peerInfo.peer_addr, peerAddress, 6);
  peerInfo.channel = 0; // 0 = use current WiFi channel
  peerInfo.encrypt = false;
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer");
  }
}

void loop() {
  SensorMessage msg = {22.5, 1};
  esp_now_send(peerAddress, (uint8_t *)&msg, sizeof(msg));
  delay(2000);
}
```

```cpp
// esp-now-receiver-esp32.ino
#include <WiFi.h>
#include <esp_now.h>

typedef struct {
  float tempC;
  int deviceId;
} SensorMessage;

void onDataRecv(const esp_now_recv_info_t *info, const uint8_t *data, int len) {
  SensorMessage msg;
  memcpy(&msg, data, sizeof(msg));
  Serial.printf("From device %d: %.1fC\n", msg.deviceId, msg.tempC);
}

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW init failed");
    return;
  }
  esp_now_register_recv_cb(onDataRecv);
}

void loop() {}
```

Finding a board's own MAC address (needed to configure the *other* side
as a peer) uses the documented `WiFi.macAddress()` call, printed once at
boot.

## Choosing between the two

UDP broadcast needs a WiFi network and router but no peer setup; ESP-NOW
needs peer MAC addresses configured ahead of time but no router at all
and lower latency. A mesh of battery sensors reporting to one gateway
node is a natural ESP-NOW fit; a "tell every device on the LAN to
refresh" announcement is a natural UDP broadcast fit.

## Exercise

1. Write the UDP sender/receiver pair and reason through what happens if
   two receivers are on the same subnet (both should get the broadcast).
2. Print `WiFi.macAddress()` on a board (conceptually) and use that value
   to configure the ESP-NOW peer array on a second board.
3. Write the ESP-NOW sender/receiver pair and explain why `WiFi.mode(WIFI_STA)`
   is required even though no `WiFi.begin()` call ever happens.
4. Explain, in a comment, one scenario where UDP broadcast's lack of
   delivery guarantee would be a problem, and one where ESP-NOW's peer
   pre-registration requirement would be inconvenient.
