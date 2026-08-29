# 08 · Connecting to WiFi

!!! note "Not flashed to hardware"
    Reasoned through against the documented `ESP8266WiFi`/`WiFi` (ESP32)
    library APIs — `WiFi.mode(WIFI_STA)`, `WiFi.begin(ssid, password)`,
    `WiFi.status()`, the `WL_CONNECTED` constant, and `WiFi.localIP()` are
    part of the stable, documented API surface shared by both cores. Not
    compiled or flashed to physical hardware in this environment — actual
    connection timing depends heavily on your router and signal strength.

## Station mode vs. access point mode

Both chips can act as a **station** (`STA`, joining an existing WiFi
network as a client — like your phone joining home WiFi) or an **access
point** (`AP`, creating its own network for other devices to join). This
module covers **station mode only**, since that's what a sensor node
reporting to your home network needs; AP mode appears later in this course
(Level 3's captive-portal module) for initial device setup without a
pre-known network to join.

## Basic connection sketch

```cpp
// wifi-station-connect.ino
#if defined(ESP8266)
  #include <ESP8266WiFi.h>
#elif defined(ESP32)
  #include <WiFi.h>
#endif

const char* WIFI_SSID = "YourNetworkName";
const char* WIFI_PASSWORD = "YourNetworkPassword";

void setup() {
  Serial.begin(115200);
  delay(100);

  WiFi.mode(WIFI_STA);           // station mode: join a network, don't host one
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Connected! IP address: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  // Connection established in setup(); nothing more to do here yet.
}
```

### Why `WiFi.mode(WIFI_STA)` first

Calling `WiFi.mode(WIFI_STA)` explicitly, before `WiFi.begin()`, avoids a
documented gotcha on both cores: if the board previously ran AP-mode or
AP+STA-mode code (or a factory-default sketch did), the radio can be left
in a combined mode that behaves unpredictably for a simple station
connection. Setting the mode explicitly at the top of `setup()` is standard
practice and costs nothing.

### The blocking `while` loop is fine here, but only here

`while (WiFi.status() != WL_CONNECTED) { delay(500); ... }` blocks
everything until the connection succeeds (or the code hangs forever if the
credentials are wrong). That's acceptable **inside `setup()`**, where
nothing else needs to run yet, but this exact pattern would be a serious
bug if placed inside `loop()` of a sketch that also needs to blink an LED,
read a button, or respond to sensors while WiFi status changes — Module 10
handles ongoing connection loss without blocking `loop()`.

## Handling connection failure gracefully

A truly production-shaped version of the connection routine should not
loop forever with no way out if the WiFi credentials are simply wrong or
the router is off:

```cpp
// wifi-connect-with-timeout.ino
#if defined(ESP8266)
  #include <ESP8266WiFi.h>
#elif defined(ESP32)
  #include <WiFi.h>
#endif

const char* WIFI_SSID = "YourNetworkName";
const char* WIFI_PASSWORD = "YourNetworkPassword";
const unsigned long WIFI_TIMEOUT_MS = 15000; // give up after 15 seconds

bool connectToWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  unsigned long startAttempt = millis();
  while (WiFi.status() != WL_CONNECTED) {
    if (millis() - startAttempt > WIFI_TIMEOUT_MS) {
      return false; // timed out
    }
    delay(300);
    Serial.print(".");
  }
  return true;
}

void setup() {
  Serial.begin(115200);
  delay(100);

  Serial.print("Connecting to WiFi");
  if (connectToWiFi()) {
    Serial.println();
    Serial.print("Connected! IP: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println();
    Serial.println("WiFi connection FAILED (timed out). Check credentials.");
    // A real project might fall back to AP mode here (see Level 3) or
    // simply retry periodically from loop() instead of hanging forever.
  }
}

void loop() {}
```

Using `millis() - startAttempt` rather than counting fixed loop iterations
is the standard non-blocking-timing idiom used throughout Arduino-style
code — it correctly handles the `millis()` counter's eventual 32-bit
rollover (roughly every 49.7 days of continuous uptime) because unsigned
integer subtraction wraps correctly even across that rollover point,
whereas comparing raw `millis()` values directly would not.

## Checking signal strength

Both cores expose `WiFi.RSSI()` (Received Signal Strength Indicator, in
dBm — closer to 0 is stronger, e.g. `-40` is excellent, `-80` is weak):

```cpp
// wifi-signal-strength.ino
void setup() {
  Serial.begin(115200);
  // ... connect first, using either sketch above ...
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    Serial.printf("RSSI: %d dBm\n", WiFi.RSSI());
  }
  delay(2000);
}
```

## Exercise

1. Fill in your own network's SSID and password in the basic connection
   sketch and confirm it prints your board's assigned local IP address.
2. Note that IP address — you'll need it again for the capstone at the end
   of this level, to open a browser to the board's web page.
3. Deliberately enter a wrong password in the timeout-aware version and
   confirm it prints the "FAILED (timed out)" message after roughly 15
   seconds rather than hanging forever.
4. Add the RSSI-printing loop to a working connection and walk your board
   (or your router, if it's portable) to a different room; observe how the
   dBm value changes and note roughly what value corresponds to a
   noticeably weaker connection in your environment.
