# 05 · Interrupts & Debouncing

!!! note "Not flashed to hardware"
    Reasoned through against the Arduino core's documented
    `attachInterrupt()` / `detachInterrupt()` API (available identically
    on ESP8266 and ESP32 cores) and the `IRAM_ATTR` requirement documented
    for ESP32 interrupt service routines. Not compiled or flashed to
    physical hardware in this environment.

## Polling vs. interrupts

Every sketch so far that reacts to a button has polled it — checking
`digitalRead()` once per `loop()` iteration. That's fine when `loop()`
runs fast and the event doesn't need microsecond-precise timing, but it
has two real weaknesses: a `loop()` busy with something else (say, an
OTA-blocking delay) can miss a fast button press entirely, and polling
constantly is wasted CPU work for an event that's rare. A **hardware
interrupt** flips this around: the chip itself watches the pin in
hardware and calls your function — the **ISR** (Interrupt Service
Routine) — the instant a specified edge occurs, regardless of what
`loop()` is doing.

## Basic `attachInterrupt()` usage

```cpp
// interrupt-basic.ino
#if defined(ESP8266)
  const int BUTTON_PIN = D5;
#elif defined(ESP32)
  const int BUTTON_PIN = 27;
#endif

// volatile is required -- the compiler must not cache this in a register,
// since the ISR modifies it asynchronously to loop()'s normal flow.
volatile bool buttonPressed = false;

// IRAM_ATTR places the ISR in internal RAM rather than flash -- the
// ESP32 core's documentation requires this for any ISR that might run
// while flash is being read/written elsewhere; ESP8266's core documents
// the same requirement via ICACHE_RAM_ATTR. Omitting it can crash the
// chip unpredictably.
#if defined(ESP32)
  void IRAM_ATTR onButtonPress() {
#else
  void ICACHE_RAM_ATTR onButtonPress() {
#endif
  buttonPressed = true;
}

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  // FALLING: button wired to GND, so pressing pulls the pin from HIGH to LOW.
  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), onButtonPress, FALLING);
}

void loop() {
  if (buttonPressed) {
    buttonPressed = false; // clear the flag before doing the (slower) work
    Serial.println("Button pressed!");
  }
}
```

Two documented rules the code above follows: the shared flag is
`volatile`, and the ISR itself does the absolute minimum (just sets a
flag) — `Serial.println()` and anything slow belongs in `loop()`, which
checks the flag, not in the ISR itself. ESP32's core documentation
explicitly warns against calling most Arduino API functions (including
`Serial` and `delay()`) from inside an ISR.

## The debouncing problem

A mechanical button's contacts don't close cleanly — they physically
bounce, generating several rapid HIGH/LOW transitions over a few
milliseconds before settling. An interrupt on `FALLING` will fire
multiple times for what a human perceives as one press, unless you
**debounce** it.

```cpp
// interrupt-debounced.ino
#if defined(ESP8266)
  const int BUTTON_PIN = D5;
#elif defined(ESP32)
  const int BUTTON_PIN = 27;
#endif

volatile bool buttonPressed = false;
volatile unsigned long lastInterruptTime = 0;
const unsigned long DEBOUNCE_MS = 50; // typical mechanical-button bounce window

#if defined(ESP32)
  void IRAM_ATTR onButtonPress() {
#else
  void ICACHE_RAM_ATTR onButtonPress() {
#endif
  unsigned long now = millis(); // millis() is documented as safe to call from an ISR
  if (now - lastInterruptTime > DEBOUNCE_MS) {
    buttonPressed = true;
    lastInterruptTime = now;
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), onButtonPress, FALLING);
}

void loop() {
  if (buttonPressed) {
    buttonPressed = false;
    Serial.println("Debounced press registered");
  }
}
```

Gating on elapsed time inside the ISR itself (rather than debouncing in
`loop()`) means every spurious bounce edge still triggers the ISR call,
but only the first one within each 50ms window sets the flag — cheap and
effective for a simple button.

## Counting events with an interrupt (e.g. a flow meter or encoder)

Interrupts are also the standard way to count fast pulses that polling
would miss entirely, such as a water flow sensor or rotary encoder:

```cpp
// interrupt-pulse-counter.ino
volatile unsigned long pulseCount = 0;

#if defined(ESP32)
  void IRAM_ATTR onPulse() {
#else
  void ICACHE_RAM_ATTR onPulse() {
#endif
  pulseCount++;
}

void setup() {
  Serial.begin(115200);
  pinMode(D6, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(D6), onPulse, RISING);
}

void loop() {
  static unsigned long lastReport = 0;
  if (millis() - lastReport > 1000) {
    lastReport = millis();
    // Snapshot with interrupts briefly disabled so the read is atomic
    // on architectures where a multi-byte read could tear mid-update.
    noInterrupts();
    unsigned long count = pulseCount;
    interrupts();
    Serial.printf("Pulses in last second: %lu\n", count);
  }
}
```

## Exercise

1. Write the basic (non-debounced) interrupt sketch and reason through
   why a single physical press could print "Button pressed!" more than
   once.
2. Add the debounce logic and explain, in a comment, why the guard
   compares against `lastInterruptTime` inside the ISR rather than in
   `loop()`.
3. Adapt the pulse-counter sketch to report pulses/second instead of a
   raw count, and explain why `noInterrupts()`/`interrupts()` bracket the
   read.
4. Identify which Arduino API calls (from prior modules) would be unsafe
   to call directly inside an ISR, and why.
