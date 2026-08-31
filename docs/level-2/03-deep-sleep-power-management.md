# 03 · Deep Sleep & Power Management

!!! note "Not flashed to hardware"
    Reasoned through against the ESP8266 Arduino core's documented
    `ESP.deepSleep(microseconds)` API and the required GPIO16-to-RST wake
    wiring, and the ESP32 Arduino core's documented
    `esp_sleep_enable_timer_wakeup()` / `esp_deep_sleep_start()` API. Not
    compiled or flashed to physical hardware in this environment — actual
    current draw depends on the specific board's onboard regulator and
    peripherals.

## Why deep sleep matters

A battery-powered sensor node that stays fully awake between readings
burns power for no reason — an ESP8266/ESP32 actively running WiFi draws
roughly 70-170 mA, while **deep sleep** documented current draw drops to
microamps to low tens of microamps depending on the exact chip. For a
node that reads a sensor once every few minutes, sleeping between reads
can be the difference between a battery lasting days versus months.

Deep sleep works by powering down almost everything (CPU, RAM contents
lost except a small RTC memory region, WiFi radio off) for a specified
duration, then triggering a full reboot — `setup()` runs again from
scratch, so state doesn't survive a sleep cycle unless you deliberately
store it in RTC memory.

## ESP8266: wiring and basic sleep

The ESP8266 requires an external wire: **GPIO16 must be physically
connected to `RST`**, because the chip's documented wake mechanism pulls
RST low itself via GPIO16 when the sleep timer expires — without that
wire, the board never wakes up.

```cpp
// deepsleep-basic-esp8266.ino
// Wiring: GPIO16 (D0 on NodeMCU) must be jumpered to RST.

void setup() {
  Serial.begin(115200);
  Serial.println("Awake, doing work...");

  // ... read sensor, publish over WiFi, etc. ...
  delay(1000); // stand-in for real work

  Serial.println("Going to sleep for 60 seconds");
  // ESP.deepSleep() takes microseconds (documented ESP8266 core API);
  // 60e6 = 60,000,000 microseconds = 60 seconds.
  ESP.deepSleep(60e6);
  // Execution never reaches here -- deepSleep() does not return.
}

void loop() {
  // Never runs -- setup() re-executes after each wake.
}
```

## ESP32: timer wakeup

ESP32's sleep API is different (and doesn't need the GPIO16 wire) — you
configure a wakeup source, then call `esp_deep_sleep_start()`:

```cpp
// deepsleep-basic-esp32.ino
#include <esp_sleep.h>

#define uS_TO_S_FACTOR 1000000ULL // microseconds per second
#define SLEEP_SECONDS 60

void setup() {
  Serial.begin(115200);
  Serial.println("Awake, doing work...");
  delay(1000);

  // esp_sleep_enable_timer_wakeup() takes microseconds (documented
  // ESP-IDF/Arduino-core API).
  esp_sleep_enable_timer_wakeup(SLEEP_SECONDS * uS_TO_S_FACTOR);
  Serial.println("Going to sleep for 60 seconds");
  esp_deep_sleep_start();
  // Execution never reaches here.
}

void loop() {}
```

## Persisting state across sleep cycles (ESP32 RTC memory)

Since `setup()` re-runs after every wake, a boot counter or last-known
value needs `RTC_DATA_ATTR` — the ESP32 Arduino core's documented
attribute that places a variable in RTC memory, which survives deep
sleep (but not a power-cycle or hard reset):

```cpp
// deepsleep-rtc-memory-esp32.ino
#include <esp_sleep.h>

RTC_DATA_ATTR int bootCount = 0; // survives deep sleep, resets on power loss

void setup() {
  Serial.begin(115200);
  bootCount++;
  Serial.printf("Boot count: %d\n", bootCount);

  // esp_sleep_get_wakeup_cause() documents which source woke the chip --
  // useful to distinguish "first power-on" from "woke from timer".
  esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
  if (cause == ESP_SLEEP_WAKEUP_TIMER) {
    Serial.println("Woke from timer sleep");
  } else {
    Serial.println("Fresh boot (power-on or reset)");
  }

  esp_sleep_enable_timer_wakeup(30 * 1000000ULL);
  esp_deep_sleep_start();
}

void loop() {}
```

## Waking on a GPIO pin (ESP32 external wakeup)

For an event-driven node (e.g. a PIR motion sensor) rather than a purely
timed one, ESP32 also documents waking on an external GPIO signal:

```cpp
// wakeup on GPIO33 going HIGH, in addition to (or instead of) a timer
esp_sleep_enable_ext0_wakeup(GPIO_NUM_33, 1); // 1 = wake on HIGH level
```

`esp_sleep_enable_ext0_wakeup()` only works with RTC-capable GPIOs (a
documented subset of ESP32 pins), so check your board's pinout before
choosing one.

## Exercise

1. On paper, wire GPIO16 to RST for an ESP8266 board and write the basic
   sleep sketch with a 30-second interval.
2. Port the same sketch to the ESP32 timer-wakeup API and add the RTC
   boot counter, confirming (by reasoning through the documented
   behavior) that it increments across wakes but would reset on a power
   cycle.
3. Combine deep sleep with the MQTT publish sketch from module 01: wake,
   connect WiFi + MQTT, publish one reading, then sleep again — this is
   the standard battery-node pattern.
4. Explain in a comment why calling `ESP.deepSleep()` inside `loop()`
   instead of `setup()` would still work but is a less common style.
