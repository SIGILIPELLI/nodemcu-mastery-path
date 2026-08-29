# 05 · Analog Input (ADC)

!!! note "Not flashed to hardware"
    Reasoned through against the documented ADC behavior in both Arduino
    cores — the ESP8266's single 10-bit `A0` channel and the ESP32's
    12-bit `analogRead` on ADC1/ADC2 pins. Not compiled or flashed to
    physical hardware in this environment. ESP32 ADC readings in
    particular are known (and documented by Espressif) to be non-linear
    near the rails — treat the numeric examples below as illustrative,
    and calibrate against a multimeter on real hardware if precision
    matters for your project.

## Why analog input exists

Digital pins only ever read `HIGH` or `LOW` — useful for buttons and
switches, useless for anything that varies continuously, like a
potentiometer's position, a light sensor's brightness, or (in Module 09) a
temperature sensor's raw voltage. An **Analog-to-Digital Converter (ADC)**
samples a voltage and reports it as a number across some resolution range.

## ESP8266: one ADC pin, 10-bit, 0–1.0 V input range

The ESP8266 has exactly **one** analog input pin, always called `A0` in the
Arduino core, with **10-bit resolution** — `analogRead(A0)` returns an
integer from `0` to `1023`.

Critically, the ESP8266's ADC pin itself only tolerates **0 to 1.0 V**
directly. Because most sensors output up to the full 3.3 V rail, **most
NodeMCU boards include an onboard voltage divider** on the `A0` pin
specifically to scale a 0–3.3 V input down into the ADC's safe 0–1.0 V
range — check your specific board's schematic, since not all ESP8266
breakout boards include this divider, and feeding more than 1.0 V into a
board without one can damage the ADC.

```cpp
// esp8266-read-potentiometer.ino
// Wire a potentiometer's wiper to A0, outer legs to 3.3V and GND.

void setup() {
  Serial.begin(115200);
}

void loop() {
  int raw = analogRead(A0);              // 0-1023 (10-bit)
  float voltage = raw * (1.0 / 1023.0);  // volts AT THE ADC PIN itself
  Serial.print("raw=");
  Serial.print(raw);
  Serial.print("  adcVoltage=");
  Serial.println(voltage, 3);
  delay(300);
}
```

## ESP32: multiple ADC pins, 12-bit, ~0–3.3 V range

The ESP32 has many more analog-capable pins, spread across two ADC
peripherals (**ADC1**, always usable; **ADC2**, shared with WiFi and
unusable while WiFi is active — avoid ADC2 pins for anything you'll read
after Module 08). Resolution is **12-bit** by default, so `analogRead()`
returns `0` to `4095`, and the input range is documented as roughly
`0`–`3.3 V` (attenuation-dependent, and non-linear near both ends per
Espressif's own ADC characterization notes).

```cpp
// esp32-read-potentiometer.ino
// Wire a potentiometer's wiper to GPIO34 (an ADC1-only input pin),
// outer legs to 3.3V and GND. GPIO34 is input-only, which is fine here.

const int POT_PIN = 34;

void setup() {
  Serial.begin(115200);
}

void loop() {
  int raw = analogRead(POT_PIN);          // 0-4095 (12-bit)
  float voltage = raw * (3.3 / 4095.0);   // approximate; ESP32 ADC is
                                           // documented as non-linear near
                                           // the rails, so treat this as
                                           // an estimate, not a calibrated
                                           // measurement
  Serial.print("raw=");
  Serial.print(raw);
  Serial.print("  approxVoltage=");
  Serial.println(voltage, 3);
  delay(300);
}
```

`GPIO34` on most ESP32 boards is an **ADC1** channel and also **input-only**
(it has no output driver at all) — a good, safe default pin to reach for
first when you just need one analog input.

## Smoothing noisy readings

Raw ADC readings jitter by a few counts even with a rock-steady input,
especially on the ESP32 where WiFi radio activity is documented to
introduce additional analog noise. A simple moving average smooths this out
without much code:

```cpp
// smoothed-analog-read.ino (ESP32 version; swap A0/34 for ESP8266)
const int ANALOG_PIN = 34;
const int SAMPLE_COUNT = 10;

int readSmoothed(int pin, int samples) {
  long total = 0;
  for (int i = 0; i < samples; i++) {
    total += analogRead(pin);
    delay(2); // small gap between samples
  }
  return total / samples;
}

void setup() {
  Serial.begin(115200);
}

void loop() {
  int smoothed = readSmoothed(ANALOG_PIN, SAMPLE_COUNT);
  Serial.println(smoothed);
  delay(200);
}
```

Averaging 10 samples trades a small amount of responsiveness (about 20 ms
of extra latency here) for meaningfully steadier numbers — a good default
ratio for slow-changing physical quantities like light level or
temperature, though it's too slow for anything that changes within a few
milliseconds.

## Exercise

1. Wire a potentiometer (or a photoresistor + fixed resistor as a voltage
   divider, if you don't have a pot) to the correct analog pin for your
   board and run the appropriate raw-read sketch above.
2. Turn the potentiometer (or cover/uncover the photoresistor) while
   watching the Serial Monitor and confirm the raw value changes smoothly
   across close to the full range.
3. Add the smoothing function to your sketch and compare the smoothed vs.
   raw values side by side — print both each loop — to see how much
   jitter the averaging removes.
4. On an ESP32 board: try reading from an ADC2 pin (e.g. GPIO2 or GPIO4,
   check your board's pinout first) while WiFi is *not* yet connected (it
   isn't until Module 08), confirm it works, and make a note to revisit
   this once you reach the WiFi module — Espressif's documentation states
   ADC2 becomes unreliable once WiFi is active, which is exactly why every
   later module in this course uses ADC1 pins only.
