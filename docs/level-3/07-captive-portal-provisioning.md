# Captive Portal & WiFi Provisioning

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP8266 `ESP8266WiFi` /
    `DNSServer` / `ESP8266WebServer` APIs and the ESP32 `WiFi` /
    `DNSServer` / `WebServer` APIs. Not compiled or flashed to physical
    hardware in this environment.

## The provisioning problem

Hard-coding a WiFi SSID and password in source works for one bench
device, not for something you'll hand to someone else. A **captive
portal** lets a device with no known network start its own access
point, serve a small web form for credentials, then reboot and join the
real network — the same pattern smart plugs and cameras use out of the
box.

## Step 1: start a SoftAP when no known network exists

```cpp
// softap-fallback.ino
#if defined(ESP8266)
  #include <ESP8266WiFi.h>
#else
  #include <WiFi.h>
#endif

const char* AP_SSID = "NodeMCU-Setup";

void startProvisioningAP() {
  WiFi.mode(WIFI_AP);
  // softAP() with no password argument creates an open AP; documented
  // on both cores. Use a password string for a protected setup AP.
  WiFi.softAP(AP_SSID);
  Serial.print("AP started, IP: ");
  Serial.println(WiFi.softAPIP()); // documented to return 192.168.4.1 by default
}
```

## Step 2: redirect every DNS query to the portal (the "captive" part)

Phones/laptops detect a captive portal by trying to resolve a
connectivity-check domain; answering *every* query with the AP's own IP
makes the OS pop the portal automatically.

```cpp
// dns-redirect.ino
#include <DNSServer.h>

DNSServer dnsServer;
const byte DNS_PORT = 53;

void startCaptiveDns() {
  IPAddress apIP = WiFi.softAPIP();
  // start(port, domain, resolvedIP): "*" documented as wildcard,
  // matches all queried domains on both cores' DNSServer library.
  dnsServer.start(DNS_PORT, "*", apIP);
}

void loop() {
  dnsServer.processNextRequest(); // must be polled, not interrupt-driven
}
```

## Step 3: serve the credentials form

```cpp
// captive-portal.ino
#if defined(ESP8266)
  #include <ESP8266WiFi.h>
  #include <ESP8266WebServer.h>
  ESP8266WebServer server(80);
#else
  #include <WiFi.h>
  #include <WebServer.h>
  WebServer server(80);
#endif
#include <DNSServer.h>
#include <LittleFS.h>

DNSServer dnsServer;
const char* AP_SSID = "NodeMCU-Setup";

const char* FORM_HTML =
  "<html><body><h3>WiFi Setup</h3>"
  "<form method='POST' action='/save'>"
  "SSID: <input name='ssid'><br>"
  "Password: <input name='pass' type='password'><br>"
  "<input type='submit' value='Save & Reboot'>"
  "</form></body></html>";

void handleRoot() {
  server.send(200, "text/html", FORM_HTML);
}

void handleSave() {
  String ssid = server.arg("ssid");
  String pass = server.arg("pass");

  File f = LittleFS.open("/wifi.cfg", "w");
  if (f) {
    f.println(ssid);
    f.println(pass);
    f.close();
  }

  server.send(200, "text/html", "Saved. Rebooting...");
  delay(1000);
  ESP.restart(); // documented on both cores
}

void setup() {
  Serial.begin(115200);
  LittleFS.begin(true);

  WiFi.mode(WIFI_AP);
  WiFi.softAP(AP_SSID);
  dnsServer.start(53, "*", WiFi.softAPIP());

  server.on("/", handleRoot);
  server.on("/save", HTTP_POST, handleSave);
  // Serve the form for any unmatched path too, since captive-portal
  // detection probes hit arbitrary URLs.
  server.onNotFound(handleRoot);
  server.begin();
}

void loop() {
  dnsServer.processNextRequest();
  server.handleClient();
}
```

`ESP8266WebServer`/`WebServer` are documented to expose the same
`on()`/`onNotFound()`/`handleClient()` surface on both cores, which is
why the sketch only needs to branch on the include and class name.

## Step 4: try the saved credentials on boot, fall back to the portal

```cpp
// boot-with-fallback.ino
bool tryStoredWifi() {
  File f = LittleFS.open("/wifi.cfg", "r");
  if (!f) return false;

  String ssid = f.readStringUntil('\n');
  String pass = f.readStringUntil('\n');
  f.close();
  ssid.trim();
  pass.trim();
  if (ssid.length() == 0) return false;

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid.c_str(), pass.c_str());

  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 15000) {
    delay(250);
  }
  return WiFi.status() == WL_CONNECTED;
}

void setup() {
  Serial.begin(115200);
  LittleFS.begin(true);

  if (!tryStoredWifi()) {
    startProvisioningAP(); // from step 1
    // ... plus DNS/web-server setup from steps 2-3
  } else {
    Serial.println("Connected: " + WiFi.localIP().toString());
  }
}
```

## Exercise

1. Explain why answering every DNS query with the AP's own IP (rather
   than only known captive-portal check domains) is what triggers most
   phones/laptops to auto-open the portal UI.
2. Extend `handleSave()` to validate the SSID field is non-empty before
   writing to LittleFS, returning a 400 response otherwise (`server.send`
   supports an arbitrary status code, per the documented API).
3. Add a "forget WiFi" mode: holding a button (GPIO input) during boot
   should call `LittleFS.remove("/wifi.cfg")` and start the
   provisioning AP even if stored credentials exist.
4. Storing WiFi passwords in a plaintext LittleFS file is common but has
   a real limitation — name it, and describe one documented mitigation
   available on the ESP32 (hint: flash encryption / NVS encryption) that
   the ESP8266 does not offer.
