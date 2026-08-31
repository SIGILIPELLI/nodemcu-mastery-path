# 03 · Multi-Sensor Data Aggregation & Buffering

!!! note "Not flashed to hardware"
    Reasoned through against the documented Arduino core APIs for the
    peripherals referenced (analog/digital reads, `millis()` timing) and
    standard C++ container semantics. Not compiled or flashed to physical
    hardware in this environment.

## From one sensor to a fleet of readings

Every earlier module read one sensor and published immediately. Real
devices usually carry several sensors (temperature, humidity, light,
vibration) sampled at different rates, and a network link that's
unreliable or expensive to wake. **Aggregation** means combining reads
into one coherent record; **buffering** means holding records locally
when the network isn't available, without losing data or exhausting RAM.

## A struct per sample, and why it matters

```cpp
// sensor-sample.h
struct SensorSample {
  uint32_t timestampMs;
  float tempC;
  float humidityPct;
  int   lightRaw;
};
```

A fixed-layout struct is cheap to copy, easy to serialize, and — crucially
on a device with 40-80 KB of usable RAM — has a predictable size you can
multiply against a buffer capacity to know exactly how much RAM a ring
buffer will cost.

## A fixed-size ring buffer (no dynamic allocation)

Heap fragmentation is a real failure mode on long-running ESP8266
sketches; a fixed-capacity ring buffer avoids `malloc`/`free` churn
entirely.

```cpp
// ring-buffer.ino
#include "sensor-sample.h"

const size_t CAPACITY = 30; // 30 * sizeof(SensorSample) ~= 30*16 = 480 bytes
SensorSample buffer[CAPACITY];
size_t head = 0;   // next write index
size_t count = 0;  // number of valid entries

void pushSample(const SensorSample &s) {
  buffer[head] = s;
  head = (head + 1) % CAPACITY;
  if (count < CAPACITY) {
    count++;
  }
  // Once full, pushSample silently overwrites the oldest entry —
  // documented ring-buffer behavior: newest data wins over oldest.
}

size_t sampleCount() {
  return count;
}

SensorSample getSample(size_t indexFromOldest) {
  size_t start = (head + CAPACITY - count) % CAPACITY;
  return buffer[(start + indexFromOldest) % CAPACITY];
}
```

## Sampling multiple sensors on independent schedules

```cpp
// multi-sensor-aggregate.ino
#include <DHT.h>
#include "sensor-sample.h"

#define DHTPIN 4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

const int LIGHT_PIN = A0;

unsigned long lastDhtRead = 0;
const unsigned long DHT_INTERVAL_MS = 2000;   // DHT11 datasheet minimum ~1-2s
unsigned long lastLightRead = 0;
const unsigned long LIGHT_INTERVAL_MS = 200;

float lastTempC = NAN;
float lastHumidity = NAN;

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  unsigned long now = millis();

  if (now - lastDhtRead >= DHT_INTERVAL_MS) {
    lastDhtRead = now;
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    if (!isnan(t)) lastTempC = t;
    if (!isnan(h)) lastHumidity = h;
  }

  if (now - lastLightRead >= LIGHT_INTERVAL_MS) {
    lastLightRead = now;
    int light = analogRead(LIGHT_PIN); // 0-1023 on ESP8266, 0-4095 on ESP32 (12-bit ADC)

    SensorSample s;
    s.timestampMs = now;
    s.tempC = lastTempC;       // most recent DHT reading, not re-sampled every tick
    s.humidityPct = lastHumidity;
    s.lightRaw = light;

    pushSample(s);
  }
}
```

The pattern: each sensor is polled at its own natural rate, but every
sample record carries the *last known* value of the slower sensors —
this is aggregation, combining data of different ages into one row
without blocking the fast loop on the slow sensor.

## Draining the buffer when the network comes back

```cpp
// drain-on-reconnect.ino
void tryFlushBuffer(WiFiClient &client) {
  if (WiFi.status() != WL_CONNECTED) return;

  while (sampleCount() > 0) {
    SensorSample s = getSample(0); // oldest first: preserve time order
    // ... build and send JSON/MQTT payload for s ...
    popOldest(); // implementation removes index 'start' and decrements count
  }
}

void popOldest() {
  if (count > 0) count--; // oldest slot is now stale; next push will overwrite it
}
```

## Exercise

1. Compute the RAM cost of a 100-entry ring buffer of `SensorSample` and
   check it against a realistic 40 KB free-heap budget.
2. Extend `multi-sensor-aggregate.ino` with a third sensor sampled every
   5 seconds, aggregated into the same `SensorSample` struct.
3. Implement `popOldest()` properly (adjusting the logical start index)
   and trace through a full fill-and-drain cycle by hand.
4. Explain why overwriting the oldest sample on overflow might be wrong
   for a data-logging use case, and sketch an alternative policy.
