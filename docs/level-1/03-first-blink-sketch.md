# 03 · First Blink Sketch

!!! note "Not flashed to hardware"
    This sketch is written and reasoned through against the documented
    ESP8266/ESP32 Arduino-core APIs (`pinMode`, `digitalWrite`, `delay`,
    the `LED_BUILTIN` constant each core defines) but has not been
    compiled or flashed to a physical board in this environment. The
    logic — pin direction, active-low behavior, timing — follows directly
    from each core's documented pin definitions; verify the exact
    on-board LED polarity for your specific board model before trusting
    the "expected behavior" notes below.

## The built-in LED, and why it can be confusing

Almost every NodeMCU and ESP32 dev board has a small LED soldered onto the
board itself, wired to a specific GPIO pin. The Arduino core for each chip
defines a constant, `LED_BUILTIN`, that points at whichever GPIO that LED is
wired to for that specific board definition — so the same line of code,
`digitalWrite(LED_BUILTIN, HIGH)`, works across different boards without you
needing to know the exact pin number.

The confusing part: **on most NodeMCU (ESP8266) boards, the built-in LED is
wired active-low** — meaning `LOW` turns it **on** and `HIGH` turns it
**off**, the opposite of what you'd naturally guess. This is a hardware
wiring choice (the LED's anode is tied to 3.3 V, cathode to the GPIO pin),
not a software quirk, and it's called out explicitly in the ESP8266 Arduino
core's board variant files. Most ESP32 dev boards wire their built-in LED
the "normal" active-high way, but not universally — always test both
polarities if the LED doesn't behave as expected.

## The sketch

```cpp
// 03-first-blink-sketch.ino
// Blinks the board's built-in LED once per second.
// Works on both ESP8266 and ESP32 Arduino-core targets.

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_BUILTIN, LOW);   // ESP8266 boards: LOW = LED on (active-low)
  delay(500);
  digitalWrite(LED_BUILTIN, HIGH);  // ESP8266 boards: HIGH = LED off
  delay(500);
}
```

### Line-by-line

- `pinMode(LED_BUILTIN, OUTPUT);` — every GPIO pin starts as an input by
  default on power-up; this call configures the built-in LED's pin as a
  push-pull digital output so `digitalWrite` can drive it.
- `digitalWrite(LED_BUILTIN, LOW);` then `delay(500);` — drives the pin low
  for 500 milliseconds. On a typical ESP8266 NodeMCU board this turns the
  LED **on** for half a second, due to the active-low wiring described
  above.
- The second `digitalWrite`/`delay` pair does the opposite, turning the LED
  **off** for the other half second, producing a steady 1 Hz blink (500 ms
  on, 500 ms off).

### If your LED blinks backwards

If you're on a board where the LED lights up when you'd expect it off (or
vice versa), swap the `LOW`/`HIGH` values in the two `digitalWrite` calls.
This is expected board-to-board variation, not a bug in your code — many
tutorials online simply assume one polarity or the other without saying so.

## Worked example: blink twice, then pause

A slightly more interesting pattern — two quick blinks, then a longer pause
— useful later as a distinct "status OK" heartbeat pattern you can visually
tell apart from a plain steady blink:

```cpp
// double-blink-heartbeat.ino
const int LED_ON = LOW;    // set to HIGH if your board's LED is active-high
const int LED_OFF = HIGH;  // set to LOW  if your board's LED is active-high

void blinkOnce(int onTimeMs) {
  digitalWrite(LED_BUILTIN, LED_ON);
  delay(onTimeMs);
  digitalWrite(LED_BUILTIN, LED_OFF);
  delay(150);
}

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
  digitalWrite(LED_BUILTIN, LED_OFF); // start with LED off
}

void loop() {
  blinkOnce(120);
  blinkOnce(120);
  delay(1200); // long pause before repeating the double-blink
}
```

Pulling the "on"/"off" logic level out into `LED_ON`/`LED_OFF` named
constants at the top means you only need to change one line if you move
this code to a board with the opposite LED polarity, instead of hunting
through every `digitalWrite` call in the sketch — a small habit worth
building early.

## Exercise

1. Upload the basic Blink sketch above to your board (**Sketch → Upload**,
   or the right-arrow button in the toolbar) and confirm the LED blinks
   once per second. If it doesn't blink, first re-check **Tools → Board**
   and **Tools → Port** from Module 02, then check for a compile error in
   the console before assuming a wiring problem — there's no external
   wiring in this exercise, just the board's own onboard LED.
2. If your LED's behavior looks inverted from the comments (lights up on
   `HIGH` instead of `LOW`), swap the two `digitalWrite` values and note
   which polarity your board actually uses — you'll need to remember this
   for every future module that uses `LED_BUILTIN`.
3. Modify the timing so the LED blinks twice as fast (250 ms on, 250 ms
   off) by changing only the two `delay()` arguments, then re-upload and
   confirm the faster blink rate visually.
4. Try the double-blink heartbeat pattern and confirm you can visually
   tell it apart from the plain single blink — this distinction becomes
   useful in later modules for signaling different device states (e.g.
   "connecting to WiFi" vs. "WiFi connected") using only the one onboard
   LED.
