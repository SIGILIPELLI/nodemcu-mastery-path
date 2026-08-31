# 04 · OTA (Over-the-Air) Updates

!!! note "Not flashed to hardware"
    Reasoned through against the ESP8266 and ESP32 Arduino cores'
    documented `ArduinoOTA` library API (`ArduinoOTA.begin()`,
    `.setHostname()`, `.setPassword()`, `.handle()`) as bundled with both
    board packages. Not compiled or flashed to physical hardware in this
    environment.

## Why OTA matters

Every sketch so far has been flashed over USB. That's fine on a desk, but
once a board is deployed — mounted in a wall, on a rooftop, or forty of
them scattered around a building — walking back over with a USB cable to
update firmware doesn't scale. **OTA (Over-the-Air) updates** let you
push new firmware to a device over WiFi instead, using the
`ArduinoOTA` library that ships with both the ESP8266 and ESP32 Arduino
cores.

## Basic ArduinoOTA sketch

```cpp
// ota-basic.ino
#include <ESP8266WiFi.h>   // swap for <WiFi.h> on ESP32
#include <ArduinoOTA.h>

const char* WIFI_SSID = "your-ssid";
const char* WIFI_PASS = "your-password";

void setup() {
  Serial.begin(115200);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println("\nWiFi connected: " + WiFi.localIP().toString());

  // setHostname() gives the device a friendly name in Arduino IDE's
  // Tools > Port network-port list, instead of a raw IP.
  ArduinoOTA.setHostname("esp-livingroom-01");

  // setPassword() is documented as required for any real deployment --
  // without it, anyone on the network can push firmware to the device.
  ArduinoOTA.setPassword("choose-a-strong-ota-password");

  ArduinoOTA.onStart([]() {
    Serial.println("OTA update starting...");
  });
  ArduinoOTA.onEnd([]() {
    Serial.println("OTA update complete, rebooting.");
  });
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });
  ArduinoOTA.onError([](ota_error_t error) {
    // ArduinoOTA documents these five error codes explicitly.
    Serial.printf("OTA Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) Serial.println("Auth Failed");
    else if (error == OTA_BEGIN_ERROR) Serial.println("Begin Failed");
    else if (error == OTA_CONNECT_ERROR) Serial.println("Connect Failed");
    else if (error == OTA_RECEIVE_ERROR) Serial.println("Receive Failed");
    else if (error == OTA_END_ERROR) Serial.println("End Failed");
  });

  ArduinoOTA.begin();
  Serial.println("OTA ready");
}

void loop() {
  // ArduinoOTA.handle() must run every loop iteration -- it's how the
  // sketch listens for an incoming update request without blocking.
  ArduinoOTA.handle();

  // ... normal sketch work continues here ...
}
```

Once flashed once over USB with this sketch running, the Arduino IDE's
**Tools → Port** menu will (per ArduinoOTA's documented mDNS
advertisement) list the device by its hostname over "Network Ports" —
selecting it and clicking Upload sends firmware over WiFi instead of
serial.

## The `loop()` blocking trap

`ArduinoOTA.handle()` only gets a chance to respond to an update request
between calls — a `loop()` with a long `delay()` (say, a sensor read that
sleeps 5 seconds) makes the device unresponsive to OTA for that whole
span. The documented fix is avoiding long blocking delays in favor of a
non-blocking `millis()`-based timing pattern:

```cpp
// ota-non-blocking-loop.ino
unsigned long lastRead = 0;
const unsigned long READ_INTERVAL = 5000;

void loop() {
  ArduinoOTA.handle(); // called every iteration, never blocked

  if (millis() - lastRead > READ_INTERVAL) {
    lastRead = millis();
    // ... read sensor, publish, etc. ...
  }
}
```

## Combining OTA with a fallback: only enabling on demand

For a battery/deep-sleep node (module 03), leaving OTA listening
constantly defeats the point of sleeping. A common documented pattern is
checking a physical button or a flag on boot and only entering an
OTA-enabled "awake and listening" mode when explicitly requested,
otherwise going straight back to sleep:

```cpp
// pseudocode sketch of the pattern -- combine with module 03's sleep code
void setup() {
  if (digitalRead(OTA_BUTTON_PIN) == LOW) {
    // hold this pin low at boot to request an OTA window instead of sleep
    setupWiFiAndOTA();
    while (true) { ArduinoOTA.handle(); delay(10); }
  } else {
    // normal sensor-and-sleep cycle
  }
}
```

## Exercise

1. Write the basic OTA sketch, flash it once (conceptually) over USB, and
   confirm it would appear under Arduino IDE's network ports by hostname.
2. Add `setPassword()` and explain, in a comment, what `OTA_AUTH_ERROR`
   means and when it would fire.
3. Convert a `delay()`-based sensor loop into the non-blocking `millis()`
   pattern so OTA stays responsive.
4. Sketch (in comments) how you'd combine this with module 03's deep
   sleep so the device only listens for OTA when a button is held at
   boot.
