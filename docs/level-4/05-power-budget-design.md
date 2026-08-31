# Power Budget Design for Battery Devices

!!! note "Not flashed to hardware"
    Reasoned through against documented ESP8266/ESP32 datasheet current
    figures and the Arduino-core deep-sleep API. Not compiled, flashed,
    or measured on physical hardware in this environment — treat the
    numeric figures as documented typical values, not measured results.

## The question a power budget answers

"Will this battery-powered sensor last the promised 6 months?" is a
math problem before it's a firmware problem: total available energy
(battery capacity) divided by average current draw gives runtime. Get
the average current wrong by not accounting for WiFi TX spikes, and a
6-month estimate becomes 3 weeks in the field.

## Step 1: know your current states

Both chips document wildly different current draw depending on radio
and CPU state:

| State | ESP8266 (typical, documented) | ESP32 (typical, documented) |
|---|---|---|
| Active, WiFi TX | ~120-170mA (spikes to ~300mA+) | ~160-240mA (spikes to ~300-400mA) |
| Active, WiFi idle/RX | ~15-20mA | ~20-30mA |
| Modem sleep (CPU on, radio off) | ~15mA | ~3-20mA |
| Light sleep | ~0.5-1mA | ~0.8mA |
| Deep sleep | ~10-20µA | ~5-10µA |

These are order-of-magnitude figures from chip vendor documentation,
not a substitute for measuring your specific board (voltage regulator
quiescent current and any onboard LEDs add to all of these).

## Step 2: model a duty cycle

A sensor that wakes, connects, publishes, and sleeps spends almost all
its time in deep sleep — the budget is dominated by how long the
"awake" portion takes, not by the sleep current.

```cpp
// duty-cycle-model.ino (illustrative constants, not measured)
// Awake phase: WiFi connect (~2-4s typical) + publish (~1s) + margin
const float AWAKE_SECONDS = 5.0;
const float AWAKE_CURRENT_MA = 180.0;   // dominated by WiFi TX
const float SLEEP_CURRENT_UA = 15.0;    // deep sleep, documented ESP8266 range

const float WAKE_INTERVAL_MINUTES = 15.0;
const float BATTERY_MAH = 2000.0;       // e.g. one 18650 cell

float estimateRuntimeDays() {
  float sleepSeconds = WAKE_INTERVAL_MINUTES * 60.0 - AWAKE_SECONDS;

  float awakeChargeMah = (AWAKE_CURRENT_MA * AWAKE_SECONDS) / 3600.0;
  float sleepChargeMah = (SLEEP_CURRENT_UA / 1000.0 * sleepSeconds) / 3600.0;

  float chargePerCycleMah = awakeChargeMah + sleepChargeMah;
  int cyclesPerDay = (24 * 60) / WAKE_INTERVAL_MINUTES;
  float dailyMah = chargePerCycleMah * cyclesPerDay;

  return BATTERY_MAH / dailyMah;
}

void setup() {
  Serial.begin(115200);
  Serial.printf("Estimated runtime: %.1f days\n", estimateRuntimeDays());
}

void loop() {}
```

Running this model (18650 at 2000mAh, wake every 15 minutes, 5s awake
at 180mA) puts the awake phase at roughly 0.00025 mAh × per-cycle
scaling — the point of the exercise isn't the exact number, it's seeing
that halving `AWAKE_SECONDS` (a firmware change: faster WiFi connect,
less work before sleep) has an outsized effect on runtime compared to
optimizing the already-tiny sleep current further.

## Step 3: the highest-leverage firmware lever — shrink the awake window

Reconnecting to WiFi from scratch (full handshake + DHCP) is the
single biggest awake-time cost on both chips. Both cores document ways
to shortcut it:

```cpp
// fast-reconnect.ino
#include <ESP8266WiFi.h> // or WiFi.h on ESP32

void connectFast() {
  WiFi.persistent(false);      // documented: avoid flash writes on every connect
  WiFi.mode(WIFI_STA);

  // Static IP skips the DHCP round-trip (documented config() overload)
  IPAddress ip(192, 168, 1, 50), gw(192, 168, 1, 1), mask(255, 255, 255, 0);
  WiFi.config(ip, gw, mask);

  WiFi.begin("ssid", "password");
  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 8000) {
    delay(50);
  }
}
```

Static IP configuration is documented on both cores as a way to skip
DHCP negotiation, which on a congested network can otherwise add
seconds to every wake cycle — seconds spent entirely at the highest
current draw state.

## Step 4: deep sleep is the other half of the lever

```cpp
// deep-sleep-cycle.ino
#if defined(ESP8266)
  #include <ESP8266WiFi.h>
#else
  #include <esp_sleep.h>
#endif

void goToSleep(uint64_t seconds) {
#if defined(ESP8266)
  // ESP.deepSleep() takes microseconds; documented max ~71 minutes
  // without external RTC modification (32-bit microsecond counter).
  ESP.deepSleep(seconds * 1000000ULL);
#else
  esp_sleep_enable_timer_wakeup(seconds * 1000000ULL); // documented ESP32 API
  esp_deep_sleep_start();
#endif
}
```

Note the ESP8266's documented ~71-minute ceiling on a single
`deepSleep()` call — a product needing longer intervals must either
chain sleep cycles or use the ESP32, whose `esp_sleep_enable_timer_wakeup`
accepts a much larger range.

## Step 5: don't forget peripheral current

Sensors and status LEDs left powered during sleep silently blow the
budget the math above doesn't account for. Documented mitigation:
switch sensor power through a GPIO-controlled transistor/MOSFET so it's
only energized during the awake window, rather than tied directly to
3.3V.

```cpp
const int SENSOR_POWER_PIN = 5;

void setup() {
  pinMode(SENSOR_POWER_PIN, OUTPUT);
  digitalWrite(SENSOR_POWER_PIN, HIGH); // power sensor only now
  delay(50); // documented sensor power-up settle time varies; check datasheet
  readSensor();
  digitalWrite(SENSOR_POWER_PIN, LOW);  // cut power before sleeping
  goToSleep(900);
}

void readSensor() {}
void loop() {}
```

## Exercise

1. Using the model function, compute estimated runtime for a 5000mAh
   battery, 30-minute wake interval, and 3-second awake window at
   150mA. Show the arithmetic.
2. Explain why static IP configuration only helps if the network's DHCP
   server (not the device) is the slow part of connection time — under
   what condition would it make no measurable difference?
3. Modify `goToSleep()` to chain multiple ESP8266 `deepSleep()` calls to
   reach a 2-hour total interval, given the ~71-minute single-call
   ceiling.
4. Propose a way to measure actual awake-phase duration in the field
   (without a bench ammeter) using only `millis()` timestamps logged
   before sleep and after wake.
