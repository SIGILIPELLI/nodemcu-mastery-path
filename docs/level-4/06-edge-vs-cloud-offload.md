# Edge Processing vs. Cloud Offload

!!! note "Not flashed to hardware"
    Reasoned through against documented ESP8266/ESP32 RAM/compute specs
    and Arduino-core APIs. Not compiled or flashed to physical hardware
    in this environment.

## The decision every sensor pipeline has to make

Raw sensor data can be sent upstream as-is, or partially processed
on-device first. The right split depends on bandwidth cost, latency
tolerance, and — on these chips specifically — how little RAM and CPU
there actually is to spend on "edge" processing.

## What the ESP8266/ESP32 can realistically do on-device

Documented RAM budgets matter here: the ESP8266 core typically leaves
tens of KB of free heap after WiFi/TLS buffers, and the ESP32
typically leaves a few hundred KB. That rules out anything resembling
a full ML model or large buffers, but comfortably fits:

- Rolling averages / min / max over a window
- Simple threshold-crossing detection
- Delta compression (send only when a value changed meaningfully)
- Basic FFT on a small sample buffer (documented `arduinoFFT` library
  works within ESP32's RAM budget for buffers in the low hundreds of
  samples)

## Pattern 1: send-on-change instead of send-every-sample

The simplest and highest-leverage edge optimization: don't publish
readings that haven't meaningfully changed.

```cpp
// send-on-change.ino
float lastSentTemp = NAN;
const float TEMP_DELTA_THRESHOLD = 0.5; // °C

void maybePublishTemp(float currentTemp) {
  bool firstReading = isnan(lastSentTemp);
  bool changedEnough = !firstReading &&
                        fabs(currentTemp - lastSentTemp) >= TEMP_DELTA_THRESHOLD;

  if (firstReading || changedEnough) {
    publishTemp(currentTemp);
    lastSentTemp = currentTemp;
  }
}

void publishTemp(float t) { /* mqtt.publish(...) */ }
```

For a slowly-changing quantity like room temperature, this can cut
publish volume by an order of magnitude with no loss of meaningful
signal, directly reducing both radio-on time (power budget module) and
cloud ingestion cost.

## Pattern 2: on-device aggregation window

Instead of publishing every 5-second sample, aggregate over a longer
window and publish a summary — trading resolution for both bandwidth
and (per Level 4.05) awake-time:

```cpp
// window-aggregation.ino
const int WINDOW_SIZE = 12; // 12 samples * 5s = 1 minute window
float samples[WINDOW_SIZE];
int sampleCount = 0;

void addSample(float value) {
  if (sampleCount < WINDOW_SIZE) {
    samples[sampleCount++] = value;
  }
  if (sampleCount == WINDOW_SIZE) {
    publishAggregate();
    sampleCount = 0;
  }
}

void publishAggregate() {
  float sum = 0, minV = samples[0], maxV = samples[0];
  for (int i = 0; i < sampleCount; i++) {
    sum += samples[i];
    if (samples[i] < minV) minV = samples[i];
    if (samples[i] > maxV) maxV = samples[i];
  }
  float avg = sum / sampleCount;
  // publish {avg, min, max} instead of 12 raw points
}
```

This turns 12 publishes into 1, at the cost of losing intra-window
detail — the right tradeoff when the consuming system only cares about
trends, not individual samples.

## Pattern 3: edge-side anomaly gating

Rather than streaming everything to the cloud for anomaly detection,
apply a cheap on-device threshold and only escalate (with higher
publish frequency, or an alert topic) when something's actually
interesting:

```cpp
// anomaly-gate.ino
const float NORMAL_MAX_TEMP = 30.0;
bool inAlertMode = false;

void handleReading(float temp) {
  bool isAnomalous = temp > NORMAL_MAX_TEMP;

  if (isAnomalous && !inAlertMode) {
    inAlertMode = true;
    publishAlert(temp); // immediate, high-priority publish
  } else if (!isAnomalous && inAlertMode) {
    inAlertMode = false;
    publishAlertCleared();
  }

  // Normal-mode readings still go through send-on-change (pattern 1);
  // alert-mode readings could switch to every-sample for the duration.
}

void publishAlert(float t) {}
void publishAlertCleared() {}
```

## When to push processing to the cloud instead

- **Cross-device correlation** (comparing readings across many
  sensors) needs data centralized anyway — no edge benefit.
  before/after comparisons across a fleet.
- **Anything requiring a model too large for the documented RAM
  budget** (most real ML beyond a tiny decision tree or linear model).
- **Auditability**: if raw readings must be retained for compliance,
  edge aggregation that discards data can't be the only copy — send
  raw to cold storage, aggregate for real-time use.

## A hybrid: buffer raw locally, publish aggregated live

Combining Level 3.06's local logging with edge aggregation gets both:
low-bandwidth live telemetry now, full-resolution data recoverable
later if needed.

```cpp
void addSample(float value) {
  appendReadingBounded(value, 0); // raw copy to LittleFS, from 3.06
  // ... plus the WINDOW_SIZE aggregation above for the live publish
}
```

## How It Actually Works

Deciding what computation runs on-device versus in the cloud is bounded by real, measurable hardware limits, not just architectural taste: these chips have tens to a few hundred KB of usable RAM and clock speeds in the 80-240MHz range without a floating-point unit on ESP8266 (floating-point math there is emulated in software, meaningfully slower than native integer math), so any on-device inference or signal-processing workload has to fit its working set inside that heap and its latency budget inside that clock speed — a computation that would be trivial cloud-side because a server has gigabytes of RAM and a multi-GHz CPU with hardware floating point can be genuinely infeasible on-device, not merely "slower," if its working set simply doesn't fit in available heap at all.

The other real constraint is radio duty cycle and its cost: every byte sent to the cloud costs actual transmit-current energy (covered in the power budget module) and consumes airtime on a shared radio channel, so edge preprocessing that reduces, say, a continuous sensor stream down to only the meaningful threshold-crossing events before transmission is directly trading available CPU cycles and RAM against radio transmit energy and network load — a legitimate case-by-case tradeoff rather than a universal "edge is better" rule, since the CPU/RAM cost of the preprocessing itself has to be smaller than the radio energy it saves to actually be worth doing on this class of hardware.

*(These examples were written and reasoned through at the register/protocol level but were not flashed to a physical board for this pass — verify timing-sensitive details against your exact chip datasheet before relying on them in production.)*

## Exercise

1. Compute the bandwidth reduction factor of send-on-change versus
   send-every-sample for a sensor that changes meaningfully once every
   20 readings on average.
2. Extend the window-aggregation pattern to also track a standard
   deviation, and explain what added value that gives a cloud-side
   dashboard over min/max/avg alone.
3. Design the alert-mode publish-frequency escalation from Pattern 3 so
   it also has a cooldown (don't flip in and out of alert mode more
   than once per minute even if the reading oscillates near the
   threshold).
4. Given the documented RAM figures for each chip, explain why an
   ESP8266-based sensor node is a worse fit than an ESP32 for the
   FFT-based edge processing pattern.
