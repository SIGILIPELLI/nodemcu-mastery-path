# 10 · Capstone — WiFi Sensor Node & Web Server

!!! note "Not flashed to hardware"
    This capstone combines the documented APIs from every earlier module —
    `WiFi.begin`/`WiFi.localIP` (Module 08), the Adafruit `DHT` library
    (Module 09), and the standard `ESP8266WebServer`/`WebServer` classes
    bundled with each Arduino core — into one sketch. It has been reasoned
    through carefully against each API's documented behavior but has not
    been compiled or flashed to physical hardware in this environment.
    Verify on a real board, and double-check the web server class name for
    your board family (they differ, as shown below) before you compile.

## What you're building

A single ESP8266 or ESP32 board that:

1. Joins your home WiFi network in station mode.
2. Reads temperature and humidity from a DHT11 sensor on a regular
   interval, without blocking anything else.
3. Prints each reading to the Serial monitor.
4. Runs a tiny built-in web server so you (or anything on your network)
   can open the board's IP address in a browser and see the latest reading
   as a simple HTML page — no separate server, phone app, or cloud service
   required.

This is deliberately the simplest possible "IoT device": no MQTT, no cloud
platform, no persistence — just local network + sensor + web page. Level 2
builds MQTT and real cloud connectivity on top of exactly this foundation.

## The built-in web server class differs by chip family

- ESP8266 Arduino core: `ESP8266WebServer` (from `<ESP8266WebServer.h>`)
- ESP32 Arduino core: `WebServer` (from `<WebServer.h>`)

Both expose the same core methods (`.on(path, handler)`, `.begin()`,
`.handleClient()`, `.send(code, contentType, body)`), which is what makes
writing one `#ifdef`-guarded sketch for both chips practical.

## Full capstone sketch

```cpp
// capstone-wifi-sensor-node.ino
#include <DHT.h>

#if defined(ESP8266)
  #include <ESP8266WiFi.h>
  #include <ESP8266WebServer.h>
  ESP8266WebServer server(80);
  const int DHT_PIN = D3;
#elif defined(ESP32)
  #include <WiFi.h>
  #include <WebServer.h>
  WebServer server(80);
  const int DHT_PIN = 27;
#endif

const char* WIFI_SSID = "YourNetworkName";
const char* WIFI_PASSWORD = "YourNetworkPassword";

DHT dht(DHT_PIN, DHT11);

// Shared state, updated periodically and read by the web handler.
// Simple globals are fine here since this sketch is single-threaded
// (loop() runs one thing at a time; there's no concurrent access).
float lastTempC = NAN;
float lastHumidity = NAN;
unsigned long lastReadMillis = 0;
const unsigned long READ_INTERVAL_MS = 5000; // read sensor every 5s

void handleRoot() {
  String html = "<!DOCTYPE html><html><head>"
                "<meta http-equiv='refresh' content='5'>" // auto-refresh
                "<title>Sensor Node</title></head><body>"
                "<h1>WiFi Sensor Node</h1>";

  if (isnan(lastTempC) || isnan(lastHumidity)) {
    html += "<p>No valid sensor reading yet.</p>";
  } else {
    html += "<p>Temperature: " + String(lastTempC, 1) + " &deg;C</p>";
    html += "<p>Humidity: " + String(lastHumidity, 1) + " %</p>";
  }

  html += "<p><small>Uptime: " + String(millis() / 1000) + " s</small></p>";
  html += "</body></html>";

  server.send(200, "text/html", html);
}

void handleNotFound() {
  server.send(404, "text/plain", "Not found");
}

bool connectToWiFi(unsigned long timeoutMs = 15000) {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED) {
    if (millis() - start > timeoutMs) return false;
    delay(300);
    Serial.print(".");
  }
  return true;
}

void setup() {
  Serial.begin(115200);
  delay(100);
  dht.begin();

  Serial.print("Connecting to WiFi");
  if (!connectToWiFi()) {
    Serial.println("\nWiFi connection failed. Restarting in 5s...");
    delay(5000);
    ESP.restart();
  }
  Serial.println();
  Serial.print("Connected. Open http://");
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);
  server.onNotFound(handleNotFound);
  server.begin();
  Serial.println("Web server started.");
}

void loop() {
  // Never call delay() here for long periods -- server.handleClient()
  // must run every loop iteration to stay responsive to incoming
  // requests, so all timing uses millis() comparisons instead.
  server.handleClient();

  unsigned long now = millis();
  if (now - lastReadMillis >= READ_INTERVAL_MS) {
    lastReadMillis = now;

    float h = dht.readHumidity();
    float t = dht.readTemperature();

    if (isnan(h) || isnan(t)) {
      Serial.println("Sensor read failed, keeping previous values.");
    } else {
      lastTempC = t;
      lastHumidity = h;
      Serial.printf("[%lus] Temp=%.1fC Humidity=%.1f%%\n",
                    now / 1000, t, h);
    }
  }
}
```

### Why this design works without ever blocking

Every earlier module either used `delay()` freely (fine when nothing else
needed to run concurrently) or explicitly called out where blocking would
become a problem. This capstone is where it finally matters: `loop()` calls
`server.handleClient()` on **every single iteration**, so an incoming
browser request is never kept waiting behind a `delay()`. The sensor read
still only happens once every 5 seconds, but it's gated by comparing
`millis()` timestamps rather than by calling `delay(5000)` — the same
non-blocking-timing idiom introduced in Module 08's timeout logic, now
doing real work instead of just waiting.

The `<meta http-equiv='refresh' content='5'>` tag in the HTML tells the
*browser* to reload the page automatically every 5 seconds, so you get a
live-updating view without writing any JavaScript or WebSocket code — a
reasonable trade-off for a first project, even though a real production
dashboard would use something more efficient (covered in Level 3's REST
API module).

## Exercise

1. Fill in your WiFi credentials, set `DHT_PIN` to match your wiring, and
   upload the full capstone sketch.
2. Watch the Serial Monitor for the "Connected. Open http://..." line and
   note the IP address.
3. Open that IP address in a web browser on a device connected to the same
   WiFi network and confirm you see the "WiFi Sensor Node" page with a
   current temperature and humidity reading.
4. Leave the page open and confirm it refreshes itself every 5 seconds with
   updated (or at least re-rendered) values, without you touching anything.
5. While the page is open in your browser, also press a button wired the
   way Module 04 described (if you still have it wired) and confirm the
   Serial Monitor still logs button-related output promptly — this proves
   `loop()` is genuinely non-blocking, since the web server and any other
   GPIO logic run smoothly side by side.
6. **Stretch goal:** add a second route, `server.on("/api", handleApi)`,
   where `handleApi` calls `server.send(200, "application/json", ...)` with
   a small hand-built JSON string containing the same temperature and
   humidity values — a preview of the proper REST API you'll build proper
   library support for in Level 3.

Completing this capstone means you've covered the entire Level 1 arc: board
setup, digital and analog I/O, PWM, Serial debugging, WiFi station mode,
and a real sensor — combined into one working, non-blocking IoT device.
Level 2 picks up from here with MQTT, cloud platforms, deep sleep, and
over-the-air updates.
