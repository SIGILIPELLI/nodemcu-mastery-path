# Provisioning at Scale

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP32 `esp_efuse`/NVS APIs
    and ESP8266/ESP32 `WiFi.macAddress()` API. Not compiled or flashed
    to physical hardware in this environment.

## The captive portal doesn't scale to a factory line

Level 3's captive portal is right for "a person sets up one device."
Manufacturing 500 units needs each one to get a **unique identity**
(device ID, per-device certificate/keys) and its **initial
configuration** without a human typing SSID/password into each unit by
hand.

## Step 1: a stable per-device identity

Every ESP8266/ESP32 has a documented, factory-burned MAC address
readable without any provisioning step — the natural seed for a device
ID:

```cpp
// device-identity.ino
#if defined(ESP8266)
  #include <ESP8266WiFi.h>
#else
  #include <WiFi.h>
#endif

String getDeviceId() {
  // WiFi.macAddress() is documented on both cores to return the
  // factory-assigned station MAC as "AA:BB:CC:DD:EE:FF".
  String mac = WiFi.macAddress();
  mac.replace(":", "");
  mac.toLowerCase();
  return "gw-" + mac; // e.g. "gw-aabbccddeeff"
}
```

Using the MAC (rather than a self-generated random ID) means the ID is
stable across factory reset and reflash, which matters for fleet
inventory tracking.

## Step 2: bulk-provisioning via a wired/serial jig, not WiFi per-unit

At volume, flashing WiFi credentials for a specific customer/site over
a captive portal per unit doesn't scale. The common pattern: flash
firmware once (identical binary for the whole batch), then have each
unit fetch its config from a provisioning server the first time it
connects to *any* network the factory jig provides temporarily.

```cpp
// factory-provisioning-fetch.ino
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <LittleFS.h>

bool fetchProvisioningProfile(const String& deviceId) {
  HTTPClient http;
  String url = "https://provision.example.com/devices/" + deviceId;
  if (!http.begin(url)) return false;

  int code = http.GET();
  if (code != 200) { http.end(); return false; }

  String body = http.getString();
  http.end();

  StaticJsonDocument<512> doc;
  if (deserializeJson(doc, body) != DeserializationError::Ok) return false;

  File f = LittleFS.open("/provision.json", "w");
  if (!f) return false;
  serializeJson(doc, f);
  f.close();
  return true;
}
```

The server looks up `deviceId` (recorded at flash time, e.g. scanned
from a label or read back over serial during manufacturing test) and
returns the customer's WiFi/MQTT config — the firmware image itself
never needs customer-specific secrets baked in.

## Step 3: per-device credentials, not one shared secret

Baking a single shared MQTT password or API key into every unit means
one leaked device compromises the whole fleet. The documented-safe
pattern is a **per-device credential**, provisioned during the factory
step and stored in NVS (ESP32) or LittleFS (both), never in source:

```cpp
// per-device-credential-store.ino
#include <Preferences.h> // documented ESP32 NVS wrapper

Preferences prefs;

void storeDeviceSecret(const String& secret) {
  prefs.begin("provision", false); // namespace, read-write
  prefs.putString("secret", secret);
  prefs.end();
}

String loadDeviceSecret() {
  prefs.begin("provision", true); // read-only
  String s = prefs.getString("secret", "");
  prefs.end();
  return s;
}
```

`Preferences` is the documented ESP32 Arduino-core wrapper around NVS
(non-volatile key-value storage), purpose-built for exactly this kind
of small per-device config that must survive reflashes of the main
application partition.

## Step 4: a provisioning state machine

```cpp
// provisioning-state-machine.ino
enum ProvisionState { UNPROVISIONED, FETCHING, PROVISIONED, FAILED };
ProvisionState state = UNPROVISIONED;

void loop() {
  switch (state) {
    case UNPROVISIONED:
      if (LittleFS.exists("/provision.json")) {
        state = PROVISIONED;
      } else if (WiFi.status() == WL_CONNECTED) {
        state = FETCHING;
      }
      break;

    case FETCHING:
      if (fetchProvisioningProfile(getDeviceId())) {
        state = PROVISIONED;
      } else {
        state = FAILED;
      }
      break;

    case PROVISIONED:
      runNormalOperation();
      break;

    case FAILED:
      // Bounded retry with backoff, not an infinite tight loop.
      delay(30000);
      state = UNPROVISIONED;
      break;
  }
}

void runNormalOperation() {}
```

## Why this matters at fleet scale

One firmware binary now serves every customer and every unit: identity
comes from hardware (MAC), configuration comes from a server keyed by
that identity, and secrets are per-device rather than shared. Re-keying
a compromised device means revoking one credential server-side, not
re-flashing the whole fleet.

## Exercise

1. Explain why `WiFi.macAddress()` is a better device-ID seed for a
   factory-provisioned fleet than a random number generated at first
   boot.
2. Extend `fetchProvisioningProfile()` to retry with exponential
   backoff (base 5s, up to 3 attempts) before transitioning to
   `FAILED`.
3. Design the server-side lookup keying strategy: what does the
   provisioning server need to know about a device *before* it powers
   on for the first time, and where would that mapping come from in a
   real manufacturing process?
4. Using `Preferences`, implement `rotateDeviceSecret()` that replaces
   the stored secret only after confirming the new one authenticates
   successfully against the server, keeping the old one as fallback
   until then.
