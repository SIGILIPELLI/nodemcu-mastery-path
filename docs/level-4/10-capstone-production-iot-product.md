# Capstone — Production IoT Product Design

!!! note "Not flashed to hardware"
    Reasoned through against every documented API used in this level
    (`esp_task_wdt`, `Preferences`/NVS, `WiFiClientSecure` with
    `setCACert`, `esp_ota_ops`, `esp_sleep`). Not compiled or flashed to
    physical hardware in this environment.

## What this capstone ties together

This is a systems design exercise: taking a battery-powered,
fleet-deployed sensor product from Level 4's individual concerns
(production boot design, provisioning at scale, fleet management, OTA
strategy, power budget, edge/cloud split, security hardening, and the
non-firmware concerns of certification and BOM cost) into one coherent
device design.

## The product: a battery-powered environmental sensor, deployed fleet-wide

Requirements: 6-month battery life on a single 18650 cell, deployed
across many sites, remotely updatable, secure against a device being
physically stolen from the field, and cheap enough in BOM to deploy at
scale.

## Combined firmware skeleton

```cpp
// production-sensor-capstone.ino
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <Preferences.h>
#include <LittleFS.h>
#include <esp_task_wdt.h>
#include <esp_sleep.h>
#include <esp_ota_ops.h>

// ---- 4.09 BOM-informed config: internal pull-up, no external resistor ----
const int WAKE_BUTTON_PIN = 4; // factory-reset trigger

// ---- 4.05 power budget: bounded awake window ----
const uint64_t SLEEP_SECONDS = 900; // 15 min
const unsigned long MAX_AWAKE_MS = 8000;

// ---- 4.07 security: real CA pinning, never setInsecure() ----
const char* ROOT_CA = "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n";

Preferences prefs;
WiFiClientSecure tlsClient;
PubSubClient mqtt(tlsClient);
String deviceId;

// ---- 4.02 identity + per-device credential ----
String getDeviceId() {
  String mac = WiFi.macAddress();
  mac.replace(":", "");
  mac.toLowerCase();
  return "sensor-" + mac;
}

String loadDeviceSecret() {
  prefs.begin("provision", true);
  String s = prefs.getString("secret", "");
  prefs.end();
  return s;
}

// ---- 4.01 production boot invariants ----
bool tryConnectWifi() {
  prefs.begin("wifi", true);
  String ssid = prefs.getString("ssid", "");
  String pass = prefs.getString("pass", "");
  prefs.end();
  if (ssid.length() == 0) return false;

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid.c_str(), pass.c_str());
  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 6000) {
    delay(100);
    esp_task_wdt_reset();
  }
  return WiFi.status() == WL_CONNECTED;
}

// ---- 4.06 edge processing: send only meaningful change ----
float lastSentTemp = NAN;
bool shouldPublish(float temp) {
  return isnan(lastSentTemp) || fabs(temp - lastSentTemp) >= 0.5;
}

// ---- 4.03 fleet management: LWT + retained state ----
bool connectMqttWithPresence() {
  tlsClient.setCACert(ROOT_CA); // 4.07: never setInsecure() in production
  mqtt.setServer("broker.example.com", 8883);

  String statusTopic = "devices/" + deviceId + "/status";
  String secret = loadDeviceSecret();
  bool ok = mqtt.connect(deviceId.c_str(), deviceId.c_str(), secret.c_str(),
                          statusTopic.c_str(), 1, true, "offline");
  if (ok) {
    mqtt.publish(statusTopic.c_str(), "online", true);
    mqtt.publish(("devices/" + deviceId + "/firmware_version").c_str(),
                 FW_VERSION, true);
  }
  return ok;
}

// ---- 4.04 OTA: staged, self-validating ----
void checkAndApplyOta() {
  const esp_partition_t* running = esp_ota_get_running_partition();
  esp_ota_img_states_t state;
  esp_ota_get_state_partition(running, &state);
  if (state == ESP_OTA_IMG_PENDING_VERIFY) {
    if (connectMqttWithPresence()) {
      esp_ota_mark_app_valid_cancel_rollback();
    } else {
      esp_ota_mark_app_invalid_rollback_and_reboot();
    }
  }
  // Actual "check for new version" HTTP call omitted here for brevity —
  // see Level 3.08 / 4.04 for the full ESPhttpUpdate flow, including
  // the staggered check-in jitter from 4.04.
}

#define FW_VERSION "2.0.0"

void factoryResetIfRequested() {
  pinMode(WAKE_BUTTON_PIN, INPUT_PULLUP); // 4.09: no external resistor needed
  if (digitalRead(WAKE_BUTTON_PIN) == LOW) {
    prefs.begin("wifi", false);
    prefs.clear();
    prefs.end();
    Serial.println("Factory reset — WiFi config cleared");
  }
}

void goToSleep() {
  esp_sleep_enable_timer_wakeup(SLEEP_SECONDS * 1000000ULL);
  esp_deep_sleep_start();
}

void setup() {
  Serial.begin(115200);
  esp_task_wdt_init(10, true);
  esp_task_wdt_add(NULL);
  LittleFS.begin(true);

  factoryResetIfRequested();
  deviceId = getDeviceId();

  checkAndApplyOta();

  unsigned long awakeStart = millis();
  if (tryConnectWifi() && connectMqttWithPresence()) {
    float temp = 21.5; // stand-in for a real sensor read
    if (shouldPublish(temp)) {
      mqtt.publish(("devices/" + deviceId + "/temp").c_str(),
                   String(temp, 1).c_str());
      lastSentTemp = temp;
    }
    mqtt.loop();
  }

  while (millis() - awakeStart < MAX_AWAKE_MS) {
    esp_task_wdt_reset();
    mqtt.loop();
    delay(10);
  }

  goToSleep();
}

void loop() {} // never reached — device sleeps at the end of setup()
```

## How each Level 4 concern shows up in this one sketch

| Concern | Where in the sketch |
|---|---|
| Production boot invariants (4.01) | Watchdog from first instruction, bounded WiFi retry |
| Provisioning at scale (4.02) | MAC-derived `deviceId`, `Preferences`-stored per-device secret |
| Fleet management (4.03) | LWT on connect, retained status/version topics |
| OTA strategy (4.04) | Self-test-then-validate via `esp_ota_ops` before doing anything else |
| Power budget (4.05) | Deep sleep between cycles, `MAX_AWAKE_MS` hard ceiling |
| Edge vs. cloud (4.06) | `shouldPublish()` send-on-change gate |
| Security hardening (4.07) | `setCACert()`, never `setInsecure()`, per-device secret auth |
| Cost engineering (4.09) | `INPUT_PULLUP` instead of an external resistor for the reset button |

Certification (4.08) has no direct code representation — it constrains
the antenna/enclosure design this firmware would ship inside, which is
exactly why it's called out in the table's absence rather than forced
into the sketch.

## Exercise

1. Add the staggered OTA check-in jitter from 4.04 so `checkAndApplyOta()`
   only performs its "check for new version" HTTP call on roughly 1 in
   6 wake cycles (spreading load across the fleet's 15-minute wake
   interval).
2. The sketch never calls `WiFi.disconnect()` or `mqtt.disconnect()`
   before sleeping — reason through whether that matters given
   `esp_deep_sleep_start()` cuts power to the radio entirely, and
   whether the broker's LWT still fires correctly in that case.
3. Estimate this device's 6-month battery life using the model from
   4.05, given `MAX_AWAKE_MS` = 8000, ~180mA awake current, and 15-minute
   wake intervals — does it meet the requirement stated at the top of
   this module?
4. Identify one place in this sketch where a bug in one Level 4 concern
   (e.g. OTA validation) could silently break another (e.g. fleet
   presence tracking), and propose a fix.
