# 01 · Building a REST API on the ESP32

!!! note "Not flashed to hardware"
    Reasoned through against the ESP32 Arduino core's documented
    `WebServer` library API (`on()`, `send()`, request-argument accessors)
    and the ESP8266 core's analogous `ESP8266WebServer` library, which
    share nearly identical documented interfaces. Not compiled or flashed
    to physical hardware in this environment.

## From a status page to a real API

Level 1's capstone served a simple HTML status page. A **REST API**
formalizes that into predictable, machine-consumable endpoints: distinct
URL paths per resource, HTTP methods (`GET` to read, `POST` to write),
and JSON request/response bodies instead of hand-built HTML — the same
shape any mobile app or dashboard expects to talk to.

## Basic GET endpoints

```cpp
// rest-api-basic.ino
#include <WiFi.h>          // ESP8266WiFi.h + ESP8266WebServer.h on ESP8266
#include <WebServer.h>
#include <ArduinoJson.h>

WebServer server(80);

float currentTempC = 22.5;

void handleStatus() {
  JsonDocument doc;
  doc["device"] = "esp-api-01";
  doc["uptimeMs"] = millis();
  doc["tempC"] = currentTempC;

  String body;
  serializeJson(doc, body);
  // send(code, contentType, body) is WebServer's documented response API.
  server.send(200, "application/json", body);
}

void handleNotFound() {
  server.send(404, "application/json", "{\"error\":\"not found\"}");
}

void setup() {
  Serial.begin(115200);
  WiFi.begin("your-ssid", "your-password");
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println(WiFi.localIP());

  server.on("/status", HTTP_GET, handleStatus);
  server.onNotFound(handleNotFound); // fires for any unmatched route
  server.begin();
}

void loop() {
  server.handleClient(); // must be called every loop iteration, non-blocking
}
```

## Accepting a POST body (JSON in, JSON out)

```cpp
// rest-api-post.ino
void handleSetInterval() {
  if (server.method() != HTTP_POST) {
    server.send(405, "application/json", "{\"error\":\"method not allowed\"}");
    return;
  }

  // server.arg("plain") is WebServer's documented way to read a raw
  // POST body when it isn't form-encoded.
  String body = server.arg("plain");

  JsonDocument doc;
  DeserializationError err = deserializeJson(doc, body);
  if (err) {
    server.send(400, "application/json", "{\"error\":\"invalid json\"}");
    return;
  }

  int intervalMs = doc["intervalMs"] | -1;
  if (intervalMs <= 0) {
    server.send(400, "application/json", "{\"error\":\"intervalMs must be positive\"}");
    return;
  }

  // ... apply intervalMs to whatever timer drives publishing ...
  server.send(200, "application/json", "{\"status\":\"ok\"}");
}

// In setup(): server.on("/config/interval", HTTP_POST, handleSetInterval);
```

## Routing with a path parameter

`WebServer` doesn't document built-in `:id`-style path parameters the way
larger frameworks do, but the same effect is achieved by registering a
prefix and parsing the remainder manually, or using
`server.pathArg()` where supported, or simplest: a query string:

```cpp
// rest-api-query-param.ino
void handleLed() {
  // /led?state=on
  String state = server.arg("state"); // arg() also reads query-string params
  if (state == "on") {
    digitalWrite(LED_BUILTIN, LOW);
  } else if (state == "off") {
    digitalWrite(LED_BUILTIN, HIGH);
  } else {
    server.send(400, "application/json", "{\"error\":\"state must be on or off\"}");
    return;
  }
  server.send(200, "application/json", "{\"status\":\"ok\"}");
}

// In setup(): server.on("/led", HTTP_GET, handleLed);
```

## CORS: letting a browser-based dashboard call the API

A dashboard hosted on a different origin (e.g. a laptop's browser, not
the ESP itself) will be blocked by the browser's same-origin policy
unless the response includes a documented CORS header:

```cpp
void handleStatus() {
  server.sendHeader("Access-Control-Allow-Origin", "*"); // dev-friendly; scope this down in production
  JsonDocument doc;
  doc["tempC"] = currentTempC;
  String body;
  serializeJson(doc, body);
  server.send(200, "application/json", body);
}
```

## Exercise

1. Write the basic `/status` GET endpoint and confirm the JSON shape it
   would return by tracing the code.
2. Add the `/config/interval` POST endpoint, including the 400 response
   for a missing/invalid `intervalMs` field.
3. Add the `/led` query-param endpoint and test (by reasoning through the
   code) both `?state=on` and an invalid `?state=maybe`.
4. Add the CORS header to every response and explain why a wildcard
   `Access-Control-Allow-Origin: *` is convenient for development but
   should be tightened for a production device.
