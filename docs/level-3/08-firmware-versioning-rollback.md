# Firmware Versioning & Rollback

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP8266
    `ESP8266httpUpdate` OTA API and the ESP32 `Update`/`esp_ota_ops`
    dual-OTA-partition model. Not compiled or flashed to physical
    hardware in this environment.

## Why "it updated" isn't enough

Level 2 covered getting new firmware onto a device over the air. This
module covers the harder half: knowing *which* version is running,
detecting a bad update before it bricks the fleet, and rolling back
automatically when it does.

## Baking a version string into every build

```cpp
// version.h
#define FW_VERSION "1.4.2"
#define FW_BUILD_DATE __DATE__ " " __TIME__
```

```cpp
// main.ino
#include "version.h"

void setup() {
  Serial.begin(115200);
  Serial.printf("Firmware %s built %s\n", FW_VERSION, FW_BUILD_DATE);
}
```

`__DATE__`/`__TIME__` are standard C preprocessor macros the Arduino
toolchain documents as available, expanding at compile time — useful
for confirming a device actually received the build you think it did.

## ESP32: dual OTA partitions give you rollback for free

The ESP32 core documents an **A/B partition scheme**: `Update.begin()`
writes to the *inactive* OTA partition, and the bootloader only
switches to it after `Update.end()` succeeds, leaving the previous,
known-good partition untouched until you explicitly commit.

```cpp
// esp32-ota-with-validation.ino
#include <Update.h>
#include <esp_ota_ops.h>

void setup() {
  Serial.begin(115200);

  // Mark this boot as "pending verify" so a crash loop rolls back
  // automatically — this pairs with app rollback enabled in
  // Tools -> "Partition Scheme" variants that include OTA + rollback,
  // or menuconfig CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE in ESP-IDF terms.
  const esp_partition_t* running = esp_ota_get_running_partition();
  esp_ota_img_states_t state;
  esp_ota_get_state_partition(running, &state);

  if (state == ESP_OTA_IMG_PENDING_VERIFY) {
    if (selfTestPasses()) {
      esp_ota_mark_app_valid_cancel_rollback(); // documented: confirms this image is good
      Serial.println("New firmware validated");
    } else {
      esp_ota_mark_app_invalid_rollback_and_reboot(); // documented: reboots into previous partition
    }
  }
}

bool selfTestPasses() {
  // Stand-in: confirm WiFi connects, sensor responds, etc.
  return true;
}

void loop() {}
```

`esp_ota_mark_app_valid_cancel_rollback()` and
`esp_ota_mark_app_invalid_rollback_and_reboot()` are documented
ESP-IDF/Arduino-core functions specifically for this "prove the new
image works before committing to it" flow.

## ESP8266: no A/B partitions, so build your own fallback

The ESP8266 core's OTA model (`ESP8266httpUpdate`) writes into a single
sketch space rather than swapping between two full partitions, so
rollback has to be modeled in application logic: keep the previous
binary's URL/version recorded, and re-flash it if the new one fails
self-test.

```cpp
// esp8266-ota-with-fallback.ino
#include <ESP8266WiFi.h>
#include <ESP8266httpUpdate.h>
#include <LittleFS.h>
#include "version.h"

void recordLastKnownGood(const String& url) {
  File f = LittleFS.open("/last-good-fw.txt", "w");
  if (f) { f.println(url); f.close(); }
}

String readLastKnownGood() {
  File f = LittleFS.open("/last-good-fw.txt", "r");
  if (!f) return "";
  String url = f.readStringUntil('\n');
  f.close();
  url.trim();
  return url;
}

void attemptUpdate(const String& newFwUrl) {
  WiFiClient client;
  t_httpUpdate_return result = ESPhttpUpdate.update(client, newFwUrl);

  switch (result) {
    case HTTP_UPDATE_OK:
      // Reboots into the new image automatically (documented behavior);
      // this line only runs if update() itself failed before rebooting.
      break;
    case HTTP_UPDATE_FAILED:
      Serial.printf("Update failed (%d): %s\n",
                     ESPhttpUpdate.getLastError(),
                     ESPhttpUpdate.getLastErrorString().c_str());
      break;
    case HTTP_UPDATE_NO_UPDATES:
      Serial.println("No update available");
      break;
  }
}

void setup() {
  Serial.begin(115200);
  LittleFS.begin(true);
  Serial.printf("Running %s\n", FW_VERSION);

  // On first boot after an update, run a self-test; if it fails,
  // re-flash the recorded last-known-good URL as a manual rollback.
  if (!selfTestPasses()) {
    String lastGood = readLastKnownGood();
    if (lastGood.length() > 0) {
      attemptUpdate(lastGood);
    }
  } else {
    recordLastKnownGood(String("https://fw.example.com/") + FW_VERSION + ".bin");
  }
}

bool selfTestPasses() { return true; }
void loop() {}
```

## A server-side gate: staged rollouts

Even with rollback, pushing a bad build to every device at once is
avoidable. A simple staging pattern: the update server checks the
device's reported version and a rollout percentage/cohort before
returning a newer binary, so a bug surfaces on 5% of the fleet instead
of 100%. `ESPhttpUpdate.update()` already sends the device's current
version via the documented `x-ESP8266-version` header (ESP8266) — a
version-aware server can use that to decide what (if anything) to serve
back, including "no update" to hold a device at the current build.

## Exercise

1. Explain why the ESP32's dual-partition OTA model gives you a
   stronger rollback guarantee than the ESP8266's single-partition
   model, in terms of what state survives a failed update.
2. Implement `selfTestPasses()` so it requires three consecutive
   successful WiFi reconnects within 60 seconds before returning true.
3. Add version comparison logic so `attemptUpdate()` is only called
   when the server's advertised version string is semantically newer
   than `FW_VERSION` (parse `MAJOR.MINOR.PATCH`).
4. Describe how you'd implement a 10%-of-fleet staged rollout using
   only the device's MAC address and a hash function, without a
   database on the server tracking cohort membership.
