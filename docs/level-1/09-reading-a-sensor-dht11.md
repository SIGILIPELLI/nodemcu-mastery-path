---
description: "Reading a Sensor (DHT11 Temperature) — The DHT11 is a combined temperature/humidity sensor that communicates over a single digital data pin using its own…"
---

# 09 · Reading a Sensor (DHT11 Temperature)

!!! note "Not flashed to hardware"
    Reasoned through against the documented public API of Adafruit's
    widely-used `DHT sensor library` (the de facto standard Arduino
    library for DHT11/DHT22 sensors, built on their single-wire digital
    protocol) — `DHT::begin()`, `DHT::readTemperature()`,
    `DHT::readHumidity()`, and the `isnan()` failure-check pattern the
    library's own examples document. Not compiled or flashed to physical
    hardware in this environment. The DHT11 in particular is documented by
    its manufacturer as slow (about 1 reading/second maximum) and coarse
    (±2°C, ±5% RH accuracy) — treat it as "good enough to learn sensor
    integration with," not as a precision instrument.

## The DHT11 sensor and its library dependency

The DHT11 is a combined temperature/humidity sensor that communicates over
a single digital data pin using its own bit-banged timing protocol — not
I2C, not SPI, not a simple analog voltage. Because that protocol involves
precise microsecond-level timing, you don't hand-roll it yourself; you use
a library that already implements it. **Adafruit's `DHT sensor library`**
is the standard choice and depends on **Adafruit's `Unified Sensor`**
library as well.

### Installing the libraries

1. **Sketch → Include Library → Manage Libraries…**
2. Search `DHT sensor library` (by Adafruit) and click **Install**. When
   prompted to also install its declared dependency, **Adafruit Unified
   Sensor**, accept — the DHT library will not compile without it.

### Wiring

A typical 3-pin DHT11 breakout module: `VCC` → 3.3V, `GND` → GND, `OUT`
(sometimes labeled `DATA` or `S`) → a digital GPIO pin of your choice. Bare
4-pin DHT11 modules (not the breakout-board version) additionally need a
10 kΩ pull-up resistor between `DATA` and `VCC` — most breakout boards
already include this resistor onboard, so check your specific module.

## Basic reading sketch

```cpp
// dht11-basic-read.ino
#include <DHT.h>

#if defined(ESP8266)
  const int DHT_PIN = D3; // GPIO0 on NodeMCU boards
#elif defined(ESP32)
  const int DHT_PIN = 27;
#endif

#define DHT_TYPE DHT11

DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  // The DHT11 is documented as needing at least ~1 second between reads;
  // reading faster than that returns stale or invalid data.
  delay(2000);

  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();      // Celsius by default
  float tempF = dht.readTemperature(true);  // pass true for Fahrenheit

  // The library's own documented convention: a failed read (bad checksum,
  // timing glitch, or nothing connected) returns NaN, not zero -- always
  // check with isnan() rather than trusting a raw numeric value.
  if (isnan(humidity) || isnan(tempC)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }

  Serial.printf("Humidity: %.1f%%  Temp: %.1fC (%.1fF)\n",
                humidity, tempC, tempF);
}
```

### Why `isnan()` matters here specifically

Unlike a digital pin (which always reads a clean `HIGH` or `LOW`) or an ADC
(which always returns *some* number even if meaningless), the DHT
library's single-wire timing protocol can fail outright — a dropped bit, a
checksum mismatch, or nothing plugged in at all — and it documents `NaN`
("Not a Number," a special floating-point value) as its explicit signal for
"this read failed, do not trust it." Skipping the `isnan()` check and using
a failed `NaN` reading directly (e.g. logging it, or averaging it into
other readings) silently corrupts everything downstream — this exact
oversight is one of the most common bugs in beginner DHT-based projects
found in online forum troubleshooting threads.

## Computing a derived value: heat index

The library also documents a `computeHeatIndex()` helper that combines
temperature and humidity into a single "feels like" number — a good
demonstration of doing something with two related sensor readings together
rather than just logging them independently:

```cpp
// dht11-heat-index.ino
#include <DHT.h>

#if defined(ESP8266)
  const int DHT_PIN = D3;
#elif defined(ESP32)
  const int DHT_PIN = 27;
#endif

DHT dht(DHT_PIN, DHT11);

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  delay(2000);

  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();

  if (isnan(humidity) || isnan(tempC)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }

  // false = compute in Celsius (matches tempC's unit)
  float heatIndexC = dht.computeHeatIndex(tempC, humidity, false);

  Serial.printf("Temp: %.1fC  Humidity: %.1f%%  Feels like: %.1fC\n",
                tempC, humidity, heatIndexC);
}
```

## A retry-on-failure pattern

Since single reads can fail intermittently even with good wiring, a small
retry loop before giving up makes a sketch noticeably more robust without
much added complexity:

```cpp
// dht11-read-with-retry.ino
#include <DHT.h>

#if defined(ESP8266)
  const int DHT_PIN = D3;
#elif defined(ESP32)
  const int DHT_PIN = 27;
#endif

DHT dht(DHT_PIN, DHT11);

bool readTemperatureWithRetry(float &outTempC, int maxAttempts = 3) {
  for (int attempt = 1; attempt <= maxAttempts; attempt++) {
    float t = dht.readTemperature();
    if (!isnan(t)) {
      outTempC = t;
      return true;
    }
    Serial.printf("DHT read attempt %d failed, retrying...\n", attempt);
    delay(2000); // respect the sensor's minimum read interval before retrying
  }
  return false;
}

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  float tempC;
  if (readTemperatureWithRetry(tempC)) {
    Serial.printf("Temperature: %.1fC\n", tempC);
  } else {
    Serial.println("DHT sensor read failed after all retries.");
  }
  delay(5000);
}
```

## How It Actually Works

The DHT11 has no SPI/I2C bus — it uses a proprietary single-wire, software-bit-banged protocol built entirely on timing. The MCU starts a transaction by pulling the data line LOW for at least 18ms (the DHT11's own power-on reset threshold) then releasing it HIGH; the sensor responds by pulling the line LOW for ~80µs, HIGH for ~80µs, then transmits 40 bits (5 bytes: humidity integer, humidity decimal, temp integer, temp decimal, checksum) by varying the *duration* of each HIGH pulse after a fixed ~50µs LOW — a short ~26-28µs HIGH encodes a 0 bit, a long ~70µs HIGH encodes a 1 bit. The Arduino DHT library reads this by busy-polling `digitalRead()` in a tight loop and measuring elapsed microseconds with `micros()` (itself backed by a free-running hardware timer/cycle counter), which is why DHT reads must run with interrupts effectively uninterrupted — a Wi-Fi radio interrupt firing mid-transaction can shift the timing enough to misread a bit, which is the real mechanism behind DHT11's notorious flakiness on ESP8266/ESP32 versus a bare AVR.

The checksum byte (sum of the four data bytes, truncated to 8 bits) is your only integrity check on a bus with no CRC/parity at the hardware level — the library discards the whole reading rather than "correcting" it because a single misread bit anywhere in the 40 shifts the physical meaning of every subsequent bit, since sensor and MCU never resynchronize mid-frame.

*(These examples were written and reasoned through at the register/protocol level but were not flashed to a physical board for this pass — verify timing-sensitive details against your exact chip datasheet before relying on them in production.)*

## Exercise

1. Install the Adafruit `DHT sensor library` and its `Unified Sensor`
   dependency, wire a DHT11 module to your chosen pin, and run the basic
   reading sketch.
2. Confirm the humidity/temperature values printed look plausible for your
   room (roughly 15–30°C and 20–70% RH indoors, in most climates).
3. Disconnect the sensor's data wire briefly while the sketch runs and
   confirm you see "Failed to read from DHT sensor!" rather than a garbage
   numeric value — this proves the `isnan()` guard is working.
4. Add the heat-index computation and log all three values (temperature,
   humidity, heat index) on one line.
5. Swap in the retry-with-backoff version and, with the sensor properly
   connected, confirm it still reads successfully on the first attempt in
   the normal case (retries should only appear if you intentionally jostle
   the wiring).
