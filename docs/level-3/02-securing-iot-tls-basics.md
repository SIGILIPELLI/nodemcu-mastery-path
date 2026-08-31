# 02 · Securing IoT Communication (TLS Basics)

!!! note "Not flashed to hardware"
    Reasoned through against the ESP8266 core's documented
    `BearSSL::WiFiClientSecure` API and the ESP32 core's `WiFiClientSecure`
    (built on mbedTLS), including their documented certificate-fingerprint
    and root-CA-pinning options. Not compiled or flashed to physical
    hardware in this environment.

## Why plain HTTP/MQTT isn't enough

Every earlier module talked to a broker or server in the clear: anyone
on the same network segment (a shared WiFi, a compromised router) can
read credentials and payloads, or inject fake responses. **TLS**
(Transport Layer Security) wraps the TCP socket in an encrypted,
authenticated channel — the same mechanism `https://` uses in a browser.
On a microcontroller this costs extra RAM and a slower handshake, so it's
worth understanding what you're actually buying and how to keep the
resource cost survivable.

## The trust problem: how does the ESP know the server is real?

TLS without certificate validation is just obfuscation — it stops casual
sniffing but not a deliberate man-in-the-middle. Two documented ways to
validate the server on these cores:

1. **Root CA pinning** — give the client the CA certificate that signed
   the server's certificate chain; the library verifies the chain
   cryptographically.
2. **Fingerprint pinning** — give the client the SHA-1 (ESP8266
   `BearSSL`) or SHA-256 fingerprint of the exact leaf certificate; it
   matches byte-for-byte but breaks the moment the server rotates its
   cert.

Fingerprinting is simpler to set up but more brittle in production;
CA pinning is the documented recommendation for anything long-lived.

## ESP32: WiFiClientSecure with a root CA

```cpp
// tls-esp32-ca.ino
#include <WiFi.h>
#include <WiFiClientSecure.h>

// PEM root CA of the server you're connecting to (e.g. your broker's CA,
// or a public CA like ISRG Root X1 for Let's Encrypt-issued servers).
const char *rootCACert = R"EOF(
-----BEGIN CERTIFICATE-----
MIIFazCCA1OgAwIBAgIRAIIQz7DSQONZRGPgu2OCiwAwDQYJKoZIhvcNAQELBQAw
... (truncated for length — paste the real PEM here) ...
-----END CERTIFICATE-----
)EOF";

WiFiClientSecure client;

void setup() {
  Serial.begin(115200);
  WiFi.begin("your-ssid", "your-password");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  // setCACert() is WiFiClientSecure's documented way to pin a trusted root.
  client.setCACert(rootCACert);

  if (client.connect("api.example.com", 443)) {
    client.println("GET /status HTTP/1.1");
    client.println("Host: api.example.com");
    client.println("Connection: close");
    client.println();
    while (client.connected() || client.available()) {
      if (client.available()) Serial.write(client.read());
    }
    client.stop();
  } else {
    Serial.println("TLS connect failed");
  }
}

void loop() {}
```

## ESP8266: BearSSL fingerprint pinning

```cpp
// tls-esp8266-fingerprint.ino
#include <ESP8266WiFi.h>
#include <WiFiClientSecureBearSSL.h>

// SHA-1 fingerprint of the server's leaf certificate, colon-separated hex.
const char fingerprint[] PROGMEM =
  "AA BB CC DD EE FF 00 11 22 33 44 55 66 77 88 99 AA BB CC DD";

BearSSL::WiFiClientSecure client;

void setup() {
  Serial.begin(115200);
  WiFi.begin("your-ssid", "your-password");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  // setFingerprint() is BearSSL::WiFiClientSecure's documented pinning API.
  client.setFingerprint(fingerprint);

  if (client.connect("api.example.com", 443)) {
    Serial.println("TLS handshake OK, fingerprint matched");
  } else {
    Serial.println("TLS connect failed (handshake or fingerprint mismatch)");
  }
}

void loop() {}
```

## Skipping validation (development only)

Both cores document an insecure escape hatch — useful while bringing up
a sketch, dangerous left in shipped firmware:

```cpp
client.setInsecure(); // ESP32 WiFiClientSecure and BearSSL both document this;
                       // disables all peer verification.
```

Treat `setInsecure()` the same as an `#ifdef DEBUG` guard: never merge it
into a build that leaves the lab.

## Budgeting RAM for the handshake

TLS session buffers are the single biggest RAM cost on these chips.
BearSSL on the ESP8266 documents a `setBufferSizes(rx, tx)` call to
shrink the default 16 KB buffers when only talking to one predictable
server (smaller buffers risk failing against servers that send larger
records):

```cpp
client.setBufferSizes(512, 512); // BearSSL-documented tuning knob;
                                  // trade generality for free heap.
```

## Exercise

1. Explain, in your own words, why `setInsecure()` protects against
   eavesdropping but not against a man-in-the-middle.
2. Write the ESP32 root-CA-pinned connect sequence for a broker at
   `mqtt.example.com:8883`.
3. Write the ESP8266 fingerprint-pinned equivalent, and describe what
   happens the day the server's certificate is renewed.
4. Add `setBufferSizes()` tuning to the ESP8266 example and explain the
   trade-off you're making.
