# 01 · What Is NodeMCU (ESP8266 vs ESP32)

!!! note "Not flashed to hardware"
    This page is conceptual — there is no code to run yet. Starting with
    the next module, every sketch is reasoned through against the
    documented ESP8266/ESP32 Arduino-core APIs, but has not been compiled
    or flashed to a physical board in this environment. Verify on real
    hardware before relying on any of it.

## What "NodeMCU" actually means

"NodeMCU" is used loosely in the hobbyist world to mean two different
things, and it helps to separate them before you buy or wire anything:

1. **NodeMCU the firmware** — an open-source Lua-based firmware project
   for the ESP8266, released in 2014. This is the *original* meaning.
2. **NodeMCU the development board** — a specific, very popular ESP8266
   dev board layout (often called "NodeMCU v2" or "NodeMCU v3" / Amica),
   with a USB-to-serial chip, voltage regulator, and breadboard-friendly
   pin headers already built in.

In modern usage — including throughout this course — "NodeMCU" almost
always means the **board**, and you almost never use the original Lua
firmware. Instead, you flash it with a C++ sketch from the **Arduino IDE**,
exactly like you would an Arduino Uno. The board just happens to be called
NodeMCU because that's the reference design most ESP8266 dev boards copied.

## The chip families: ESP8266 vs. ESP32

Both are made by Espressif and both are the workhorses of low-cost DIY IoT.
They are programmed the same way (Arduino IDE, C++, `setup()`/`loop()`), but
they are not the same chip.

| | ESP8266 | ESP32 |
|---|---|---|
| CPU | Single-core, ~80/160 MHz | Dual-core (most variants), up to 240 MHz |
| RAM | ~50 KB usable | ~320 KB+ |
| WiFi | 2.4 GHz 802.11 b/g/n | 2.4 GHz 802.11 b/g/n |
| Bluetooth | None | Classic BT + BLE (most variants) |
| GPIO pins | ~11 usable | ~25+ usable |
| ADC | 1 channel, 10-bit, 0–1.0 V (board-dependent divider to 3.3 V) | Multiple channels, 12-bit, on ADC1/ADC2 |
| Hardware PWM | Software-emulated PWM on any pin | True hardware LEDC PWM peripheral |
| Typical dev board | NodeMCU, Wemos D1 Mini | ESP32 DevKitC, NodeMCU-32S, Wemos LOLIN32 |
| Good first project fit | Simple sensor nodes, low pin-count projects | Anything needing more pins, BLE, or more RAM/CPU |

Practical rule of thumb for this course: if a module says "works on both,"
the code targets the common Arduino-core API surface (`digitalWrite`,
`WiFi.begin`, etc.) that both cores implement, with small differences called
out explicitly (most commonly around PWM and ADC, which differ the most
between the two families).

## Board packages, not "the chip itself"

You never program the bare ESP8266/ESP32 chip directly in this course — you
install a **board support package** ("core") into the Arduino IDE that knows
how to compile Arduino-style sketches for that chip and knows the exact pin
layout of your board variant. The next module walks through installing
both packages side by side, since it's common to own one of each.

## Choosing pin numbers: the GPIOxx vs Dxx trap

This is the single most common source of "why doesn't my pin do anything"
confusion for beginners, so it's worth calling out here before you write any
code.

- On the **ESP8266 NodeMCU board**, the silkscreen prints labels like `D0`,
  `D1`, `D2` … but these are **not** the same as the underlying GPIO number.
  For example, the silkscreen `D4` is actually `GPIO2`, and `D2` is
  `GPIO4`. The ESP8266 Arduino core defines friendly constants (`D0`, `D1`,
  … `D8`) that map to the correct `GPIO` numbers for the *NodeMCU board
  layout specifically* — so `digitalWrite(D4, HIGH)` does the right thing
  as long as you selected a NodeMCU-compatible board in **Tools → Board**.
- On the **ESP32 DevKitC-style board**, the silkscreen usually prints the
  actual GPIO number directly (e.g. `GPIO2`, `GPIO4`), and you write
  `digitalWrite(2, HIGH)` or `digitalWrite(GPIO_NUM_2, HIGH)` — no `Dxx`
  aliasing layer exists by default for most ESP32 boards.

Because of this, code samples in this course that need to run identically on
both families use a `LED_BUILTIN` constant (defined by each board package)
or explicit `#ifdef ESP8266` / `#ifdef ESP32` blocks rather than hardcoding
`D4`.

## A note on power and USB cables

Both boards run their logic at **3.3 V**, not 5 V, even though the USB port
supplies 5 V (regulated down on-board). Two practical consequences that
trip up beginners:

- GPIO pins are **not 5 V tolerant** in most cases — feeding a 5 V sensor
  output straight into a GPIO pin can damage it. Use a level shifter or
  voltage divider for 5 V sensors.
- Many USB cables are "charge-only" (power wires only, no data wires). If
  the Arduino IDE never sees a serial port appear when you plug the board
  in, try a different, known-good data-capable USB cable before assuming
  the board or drivers are broken — this single mistake wastes more
  beginner time than any driver issue.

## Exercise

No code yet — this is a reading/setup-decision module. Before moving on,
write down (on paper or in a notes file) the answers to these three
questions, based on the board(s) you actually have or plan to buy:

1. Is your board an ESP8266-family board (e.g. NodeMCU, Wemos D1 Mini) or
   an ESP32-family board (e.g. ESP32 DevKitC, NodeMCU-32S)? Look for the
   chip package printed on the metal-can module soldered to the board.
2. Does the silkscreen print `Dxx` labels (a strong signal it's an ESP8266
   NodeMCU-layout board) or `GPIOxx`/plain numbers (typical of ESP32
   boards)?
3. Do you have a USB cable you've already confirmed passes data (not just
   power) — for example, one you've used to sync a phone or transfer files?

You'll need these answers in the very next module, where you install the
correct board package and pick the right entry in **Tools → Board**.
