# Task Scheduling on a Single Core

!!! note "Not flashed to hardware"
    Reasoned through against the documented Arduino core `millis()`
    timer model and the ESP8266/ESP32 `Ticker` library API. Not compiled
    or flashed to physical hardware in this environment.

## The problem with `delay()`

A single-core sketch has exactly one `loop()`. Every `delay()` call
blocks that loop entirely — no sensor reads, no MQTT keepalive, no web
server response can happen while it sleeps. Real firmware needs to run
several logical "tasks" (read a sensor every 2s, publish every 30s,
blink a status LED every 500ms) concurrently on that one loop, which
means replacing `delay()` with **non-blocking, time-sliced scheduling**.

## Pattern 1: independent `millis()` timers

The simplest scheduler is a set of "last run" timestamps checked each
pass through `loop()`:

```cpp
// multi-task-millis.ino
unsigned long lastSensorRead = 0;
unsigned long lastPublish    = 0;
unsigned long lastBlink      = 0;

const unsigned long SENSOR_INTERVAL_MS  = 2000;
const unsigned long PUBLISH_INTERVAL_MS = 30000;
const unsigned long BLINK_INTERVAL_MS   = 500;

const int LED_PIN = LED_BUILTIN;
bool ledState = false;

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  unsigned long now = millis();

  if (now - lastSensorRead >= SENSOR_INTERVAL_MS) {
    lastSensorRead = now;
    readSensor();
  }

  if (now - lastPublish >= PUBLISH_INTERVAL_MS) {
    lastPublish = now;
    publishData();
  }

  if (now - lastBlink >= BLINK_INTERVAL_MS) {
    lastBlink = now;
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
  }

  // No delay() here — loop() returns quickly, keeping every
  // interval responsive and feeding the watchdog implicitly.
}

void readSensor()  { /* ... */ }
void publishData() { /* ... */ }
```

Subtracting `unsigned long` timestamps (`now - lastX`) is documented as
safe across the `millis()` rollover at ~49.7 days because unsigned
integer wraparound arithmetic still produces the correct elapsed value —
comparing with `>` directly (`now > lastX + interval`) is not safe for
the same reason and should be avoided.

## Pattern 2: a small table-driven scheduler

Once you have more than three or four tasks, a table keeps the logic
declarative instead of one `if` per task:

```cpp
// task-table-scheduler.ino
struct Task {
  void (*fn)();
  unsigned long intervalMs;
  unsigned long lastRun;
};

void taskReadSensor();
void taskPublish();
void taskBlink();

Task tasks[] = {
  { taskReadSensor, 2000,  0 },
  { taskPublish,    30000, 0 },
  { taskBlink,      500,   0 },
};
const int NUM_TASKS = sizeof(tasks) / sizeof(tasks[0]);

void setup() {
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  unsigned long now = millis();
  for (int i = 0; i < NUM_TASKS; i++) {
    if (now - tasks[i].lastRun >= tasks[i].intervalMs) {
      tasks[i].lastRun = now;
      tasks[i].fn();
    }
  }
}

void taskReadSensor() { /* ... */ }
void taskPublish()    { /* ... */ }
void taskBlink() {
  static bool state = false;
  state = !state;
  digitalWrite(LED_BUILTIN, state);
}
```

## Pattern 3: the `Ticker` library

Both cores ship a documented `Ticker` class that wraps timer-driven
callbacks so you don't hand-roll the `millis()` bookkeeping:

```cpp
// ticker-scheduler.ino
#include <Ticker.h>

Ticker sensorTicker;
Ticker blinkTicker;
volatile bool sensorDue = false;

void onSensorTick() {
  // Ticker callbacks run in interrupt-adjacent context on both cores
  // (documented as timer-ISR-driven) — keep them tiny; just set a flag.
  sensorDue = true;
}

void onBlinkTick() {
  digitalWrite(LED_BUILTIN, !digitalRead(LED_BUILTIN));
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);
  sensorTicker.attach(2.0, onSensorTick);   // seconds, repeating
  blinkTicker.attach_ms(500, onBlinkTick);  // milliseconds, repeating
}

void loop() {
  if (sensorDue) {
    sensorDue = false;
    readSensor(); // do the real (slower) work safely in loop(), not the callback
  }
}

void readSensor() { /* ... */ }
```

`attach()`/`attach_ms()` and the flag-then-handle-in-loop() pattern are
the documented safe way to use `Ticker`: because the callback executes
in a timer-interrupt-like context, calling anything non-trivial
(`Serial.print`, network calls, `delay()`) directly inside it is
documented as unsafe on both cores.

## Choosing between the patterns

- **millis() timers** — no dependencies, most portable, best for 2-4
  tasks.
- **Task table** — same mechanism, better as the task count grows.
- **Ticker** — best when a task must fire precisely regardless of how
  long other loop() work takes, at the cost of writing interrupt-safe
  callbacks.

## Exercise

1. Convert a sketch that currently uses three stacked `delay()` calls
   (500ms, 1000ms, 5000ms) into the millis()-timer pattern above.
2. Extend the task-table scheduler with a fourth task that runs once
   every 10 minutes, and explain why `unsigned long` is required for
   the interval and timestamp fields.
3. Rewrite the `Ticker` example so the sensor read happens directly in
   `onSensorTick()`, and explain from the documented API constraints why
   that is a bad idea if `readSensor()` calls `Serial.print()` or blocks.
4. Describe a scenario where a table-driven scheduler task can starve
   another task even though neither calls `delay()`.
