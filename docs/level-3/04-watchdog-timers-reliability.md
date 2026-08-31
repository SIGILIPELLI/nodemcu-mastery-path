# 04 · Watchdog Timers & Reliability

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP8266 core software
    watchdog (`ESP.wdtEnable`/`ESP.wdtFeed` and its always-on hardware
    watchdog) and the ESP32 core's `esp_task_wdt` API. Not compiled or
    flashed to physical hardware in this environment.

## Devices hang. Watchdogs recover them.

A device left in the field will eventually hit a bug you didn't
reproduce on the bench: a library blocking forever, a heap allocation
failure, a sensor that stops acknowledging on I2C. Without a way to
detect "the main loop stopped responding," the only fix is a human
pulling power. A **watchdog timer (WDT)** is a hardware countdown that
resets the chip unless the firmware "feeds" it regularly — silence past
the timeout means something went wrong, so it reboots you back to a
known state.

## ESP8266: hardware WDT is always running

The ESP8266 documents a hardware watchdog that's active by default and
resets the chip (~8s timeout) if `loop()` never returns control to the
core — this alone catches most infinite loops for free. The core also
documents a **software watchdog** layered on top, tunable with
`ESP.wdtEnable()`/`ESP.wdtFeed()`:

```cpp
// esp8266-watchdog.ino
void setup() {
  Serial.begin(115200);
  // wdtEnable() with no interval argument uses the documented default;
  // this sketch relies mostly on the always-on hardware WDT.
}

void loop() {
  doWork();
  yield(); // documented as feeding both the software and hardware WDT;
           // required in any loop with a long-running block
}

void doWork() {
  for (int i = 0; i < 1000; i++) {
    // Any loop long enough to starve the WDT needs an explicit yield()
    // or ESP.wdtFeed() inside it, not just at the top of loop().
    if (i % 100 == 0) yield();
    delayMicroseconds(500);
  }
}
```

## ESP32: the task watchdog API

The ESP32 core documents `esp_task_wdt_init/add/reset` for supervising
specific FreeRTOS tasks (the Arduino `loop()` runs as one such task):

```cpp
// esp32-watchdog.ino
#include <esp_task_wdt.h>

const int WDT_TIMEOUT_S = 5;

void setup() {
  Serial.begin(115200);

  // esp_task_wdt_init(timeout_s, panic_on_timeout) is the documented
  // ESP32 core API; panic=true reboots on timeout.
  esp_task_wdt_init(WDT_TIMEOUT_S, true);
  esp_task_wdt_add(NULL); // NULL registers the calling task (loop's task)
}

void loop() {
  doWork();
  esp_task_wdt_reset(); // documented "feed" call; must run within the timeout window
}

void doWork() {
  // Simulate a bounded unit of work per loop iteration.
  delay(50);
}
```

## Catching a hang mid-block, not just at loop's top

A single slow call buried inside `doWork()` can still starve the
watchdog if nothing resets it until `doWork()` returns. Feed from inside
long-running sections too:

```cpp
void doLongOperation() {
  for (int i = 0; i < 200; i++) {
    processChunk(i);
    if (i % 20 == 0) {
      esp_task_wdt_reset(); // ESP32
      // yield();           // ESP8266 equivalent
    }
  }
}
```

## Detecting *why* the last reset happened

Both cores document a reset-reason API, useful for logging "was this a
watchdog recovery or a normal power-up?":

```cpp
// ESP8266
#include <ESP8266WiFi.h>
Serial.println(ESP.getResetReason()); // e.g. "Watchdog", "Power on", "External System"

// ESP32
#include <esp_system.h>
esp_reset_reason_t reason = esp_reset_reason();
if (reason == ESP_RST_TASK_WDT || reason == ESP_RST_WDT) {
  Serial.println("Recovered from watchdog reset");
}
```

Publishing this on boot (as part of the status payload from Level 1/2's
capstones) turns silent field reboots into a diagnosable signal instead
of a mystery gap in your data.

## A software "deadman" check as a second line of defense

For logic that legitimately calls slow libraries (e.g. flash writes),
pair the hardware WDT with an application-level check that something
useful is *actually* progressing, not just that `loop()` is spinning:

```cpp
unsigned long lastGoodPublish = 0;
const unsigned long MAX_SILENCE_MS = 60000;

void loop() {
  if (publishStatus()) {
    lastGoodPublish = millis();
  }
  if (millis() - lastGoodPublish > MAX_SILENCE_MS) {
    Serial.println("No successful publish in 60s — forcing restart");
    ESP.restart(); // documented on both cores
  }
  esp_task_wdt_reset(); // or yield() on ESP8266
}
```

## Exercise

1. Explain the difference between the ESP8266's hardware WDT and its
   software WDT, and why the hardware one alone doesn't guarantee your
   application logic is healthy.
2. Write the ESP32 task-watchdog setup with a 3-second timeout and a
   `loop()` that legitimately takes 2 seconds per iteration.
3. Add reset-reason logging to a `setup()` on each core and describe
   what value each would report after a watchdog-triggered reboot.
4. Implement the "deadman" check pattern around a hypothetical
   `sendTelemetry()` function that can silently fail without throwing.
