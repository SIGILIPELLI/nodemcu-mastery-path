# 04 · GPIO Basics: Digital Read & Write

!!! note "Not flashed to hardware"
    Reasoned through against the documented `pinMode`/`digitalRead`/
    `digitalWrite` behavior and internal pull-up/pull-down support in the
    ESP8266 and ESP32 Arduino cores. Not compiled or flashed to physical
    hardware in this environment — confirm pin numbering and pull-up
    availability against your exact board's pinout diagram before wiring.

## Digital output: driving an external LED

Module 03 used the board's built-in LED. This module wires up an external
LED on a general-purpose pin, which is what you'll do for almost every real
project.

**Wiring:** LED anode (long leg) → a current-limiting resistor (220–330 Ω)
→ GPIO pin. LED cathode (short leg) → GND.

```cpp
// external-led-blink.ino
// ESP8266 NodeMCU: use a Dxx pin, e.g. D1 (which is GPIO5).
// ESP32 DevKitC:   use a plain GPIO number, e.g. GPIO 25.

#if defined(ESP8266)
  const int LED_PIN = D1;
#elif defined(ESP32)
  const int LED_PIN = 25;
#endif

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH); // LED on (this LED is wired active-high)
  delay(300);
  digitalWrite(LED_PIN, LOW);  // LED off
  delay(300);
}
```

Unlike the built-in LED in Module 03, an LED you wire yourself is under your
control — wire the cathode to GND and the anode (through the resistor) to
the GPIO pin, and `HIGH` = on works exactly as you'd expect, no inversion.

## Digital input: reading a pushbutton

Buttons need a *defined* voltage level when not pressed, or the pin will
"float" and read random noise. The clean way to do this without any extra
external resistor is the microcontroller's **internal pull-up resistor**.

**Wiring:** one leg of the button → GPIO pin. Other leg → GND. No external
resistor needed.

```cpp
// button-read-pullup.ino
#if defined(ESP8266)
  const int BUTTON_PIN = D2; // GPIO4
#elif defined(ESP32)
  const int BUTTON_PIN = 26;
#endif

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
}

void loop() {
  int state = digitalRead(BUTTON_PIN);
  if (state == LOW) {
    // Pulled LOW means the button is bridging the pin to GND: pressed.
    Serial.println("Button pressed");
  }
  delay(50); // simple polling delay; see the debouncing module in Level 2
             // for a more robust, interrupt-driven approach
}
```

### Why `INPUT_PULLUP` reads `LOW` when pressed

`INPUT_PULLUP` enables an internal resistor that weakly pulls the pin
toward 3.3 V (logic `HIGH`) when nothing else is connected. Wiring the
button between the pin and GND means: **button not pressed** → pin floats
high via the pull-up → reads `HIGH`. **Button pressed** → pin is now
directly shorted to GND, overpowering the weak pull-up → reads `LOW`. This
inverted-feeling logic (pressed = LOW) is the standard, recommended pattern
on both ESP8266 and ESP32 because it needs zero external components.

!!! warning "ESP8266 pin restrictions"
    Not every ESP8266 GPIO supports `INPUT_PULLUP` the same way, and pin
    `D0` (`GPIO16`) in particular has no internal pull-up at all — it has a
    pull-*down* instead and needs special handling. Stick to `D1`–`D7` for
    simple button inputs while learning, and consult your board's pinout
    diagram for exceptions.

## Combining both: an LED that follows a button

```cpp
// button-controls-led.ino
#if defined(ESP8266)
  const int BUTTON_PIN = D2;
  const int LED_PIN    = D1;
#elif defined(ESP32)
  const int BUTTON_PIN = 26;
  const int LED_PIN    = 25;
#endif

void setup() {
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  bool pressed = (digitalRead(BUTTON_PIN) == LOW);
  digitalWrite(LED_PIN, pressed ? HIGH : LOW);
}
```

Notice there's no `delay()` in `loop()` here at all — reading a pin and
writing a pin are both effectively instantaneous, so the LED tracks the
button in real time with no perceptible lag. This "read input, react
immediately, no delay" pattern is the foundation every later module builds
on, including the WiFi and sensor modules where blocking `delay()` calls
become actively harmful.

## Exercise

1. Wire an external LED (with resistor) to a GPIO pin of your choice and
   adapt the pin constant in the first sketch to match; confirm it blinks.
2. Wire a pushbutton between a GPIO pin and GND (no resistor) and run the
   `INPUT_PULLUP` sketch; open the Serial Monitor (next module covers this
   in depth, but you can open it now via **Tools → Serial Monitor**, baud
   rate `115200`) and confirm "Button pressed" appears each time you press
   the button.
3. Combine both sketches into the button-controls-LED version and confirm
   the LED lights only while the button is held down.
4. Modify the combined sketch so the LED **toggles** on each button press
   (stays on after you release the button, until you press again) rather
   than following the button state directly. Hint: you'll need a variable
   that remembers the LED's current state across loop iterations, and you
   need to detect the *transition* from not-pressed to pressed (an "edge"),
   not just the pressed state itself — otherwise it will toggle many times
   during a single press due to how fast `loop()` runs.
