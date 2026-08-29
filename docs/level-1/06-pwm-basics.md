# 06 · PWM Basics

!!! note "Not flashed to hardware"
    Reasoned through against the documented PWM implementations in both
    cores: the ESP8266's software-emulated `analogWrite` (default 10-bit,
    ~1 kHz) and the ESP32's hardware **LEDC** peripheral (used via the
    `ledcAttach`/`ledcWrite` API in current ESP32 Arduino-core releases).
    Not compiled or flashed to physical hardware in this environment —
    the ESP32 LEDC API in particular has changed across Arduino-ESP32
    core versions, so check your installed core version's docs if a
    function signature doesn't match exactly.

## What PWM is, briefly

Pulse-Width Modulation switches a digital pin on and off very fast and
varies the fraction of time it's on (the "duty cycle") to simulate an
analog output — a 50% duty cycle looks like "half brightness" to an LED or
"half speed" to some motors, even though the pin is technically always
either fully `HIGH` or fully `LOW`.

## ESP8266: `analogWrite`, software-emulated

The ESP8266 Arduino core provides `analogWrite(pin, value)` on (almost) any
GPIO pin. Internally this is a **software-emulated** PWM (there's no
dedicated PWM hardware peripheral being used here the way there is on
ESP32) — the range and default frequency are documented, and you can
control both:

```cpp
// esp8266-pwm-fade.ino
#if defined(ESP8266)
  const int LED_PIN = D1; // any GPIO works with analogWrite on ESP8266
#endif

void setup() {
  // Default analogWrite range on ESP8266 is 0-1023 (10-bit), unlike the
  // classic AVR Arduino's 0-255 -- easy to trip over if you're used to
  // Uno-style code.
}

void loop() {
  for (int duty = 0; duty <= 1023; duty += 5) {
    analogWrite(LED_PIN, duty);
    delay(5);
  }
  for (int duty = 1023; duty >= 0; duty -= 5) {
    analogWrite(LED_PIN, duty);
    delay(5);
  }
}
```

The default `analogWrite` range on ESP8266 is `0`–`1023`, not the `0`–`255`
range classic AVR-based Arduino boards use — code copied from an Uno
tutorial that assumes 8-bit PWM will visibly under-brighten an LED on
ESP8266 unless you rescale it.

## ESP32: hardware LEDC peripheral

The ESP32 has a dedicated **LEDC** (LED Control) hardware peripheral with
multiple independent channels, each with configurable frequency and
resolution — genuinely different hardware from the ESP8266's software
emulation, not just a different function name. Current Arduino-ESP32 cores
expose it through `ledcAttach()` + `ledcWrite()`:

```cpp
// esp32-pwm-fade.ino
const int LED_PIN = 25;
const int PWM_FREQ_HZ = 5000;   // 5 kHz carrier frequency
const int PWM_RESOLUTION_BITS = 8; // 0-255 duty range at this resolution

void setup() {
  // Attaches the LEDC peripheral to this pin at the given frequency and
  // bit resolution. The core auto-assigns a free LEDC channel internally
  // in recent Arduino-ESP32 releases.
  ledcAttach(LED_PIN, PWM_FREQ_HZ, PWM_RESOLUTION_BITS);
}

void loop() {
  for (int duty = 0; duty <= 255; duty += 3) {
    ledcWrite(LED_PIN, duty);
    delay(5);
  }
  for (int duty = 255; duty >= 0; duty -= 3) {
    ledcWrite(LED_PIN, duty);
    delay(5);
  }
}
```

!!! warning "Older ESP32 core versions used a different API"
    Arduino-ESP32 core versions before 3.x used an explicit-channel API:
    `ledcSetup(channel, freq, resBits)` followed by
    `ledcAttachPin(pin, channel)` and `ledcWrite(channel, duty)` (writing
    by *channel number*, not by pin). If `ledcAttach(pin, freq, bits)`
    fails to compile on your installed core, you likely have an older core
    version — check **Tools → Board → Boards Manager** for your installed
    "esp32" package version and consult that version's migration notes.

## Using PWM for something other than LED brightness: a simple "breathing" effect

A non-linear fade — using an easing curve instead of a straight linear
ramp — looks noticeably more natural to the eye, since human brightness
perception isn't linear either:

```cpp
// breathing-led-esp32.ino
#include <math.h>

const int LED_PIN = 25;

void setup() {
  ledcAttach(LED_PIN, 5000, 8);
}

void loop() {
  for (int i = 0; i < 360; i++) {
    // Map a sine wave (0..1) onto the 0-255 duty range for a smooth,
    // natural-looking "breathing" brightness curve.
    float phase = i * (PI / 180.0);
    float level = (sin(phase) + 1.0) / 2.0; // 0.0 .. 1.0
    ledcWrite(LED_PIN, (int)(level * 255));
    delay(8);
  }
}
```

## Exercise

1. Wire an LED (with resistor) to a PWM-capable pin and run the fade sketch
   matching your board family; confirm it fades smoothly up and down.
2. On ESP8266: change the loop's `duty += 5` step to `duty += 100` and
   observe how much choppier the fade looks with fewer intermediate
   brightness steps — this demonstrates why the 10-bit range gives smoother
   fades than 8-bit would.
3. On ESP32: change `PWM_FREQ_HZ` from `5000` to `50` and observe (and
   possibly hear, as an audible whine from some LED/resistor/wiring
   combinations) the difference a much lower carrier frequency makes —
   this is the same principle used later to drive hobby servos, which
   expect a specific ~50 Hz control signal rather than a high-frequency one.
4. Try the sine-wave "breathing" sketch (ESP32) or adapt it to
   `analogWrite` (ESP8266, swapping the 0-255 output onto the 0-1023 range)
   and compare how it looks against the plain linear fade from step 1.
