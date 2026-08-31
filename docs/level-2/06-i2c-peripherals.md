# 06 · I2C Peripherals

!!! note "Not flashed to hardware"
    Reasoned through against the Arduino core's documented `Wire`
    library API (`Wire.begin()`, `Wire.beginTransmission()`,
    `Wire.requestFrom()`) shared by ESP8266 and ESP32, and Adafruit's
    documented `Adafruit_BME280`/`Adafruit_SSD1306` library APIs as
    representative I2C peripheral drivers. Not compiled or flashed to
    physical hardware in this environment.

## What I2C is and why it's everywhere

**I2C** ("Inter-Integrated Circuit") is a two-wire bus — `SDA` (data) and
`SCL` (clock) — that lets a microcontroller talk to many peripheral chips
using only those two shared wires, with each chip distinguished by a
7-bit **address**. It's the standard bus for small sensors, OLED
displays, RTCs (real-time clocks), and port expanders because wiring
stays simple even with several devices on the same bus.

On NodeMCU-style ESP8266 boards, the default I2C pins are whatever you
pass to `Wire.begin(sda, scl)` — commonly `D2` (GPIO4, SDA) and `D1`
(GPIO5, SCL). ESP32 boards default to GPIO21 (SDA) / GPIO22 (SCL) but
`Wire.begin()` accepts explicit pins there too.

## Scanning the bus

Before wiring blind, an **I2C scanner** sketch (a well-known community
pattern built entirely from the documented `Wire` API) finds what
addresses are actually present:

```cpp
// i2c-scanner.ino
#include <Wire.h>

#if defined(ESP8266)
  const int SDA_PIN = D2;
  const int SCL_PIN = D1;
#elif defined(ESP32)
  const int SDA_PIN = 21;
  const int SCL_PIN = 22;
#endif

void setup() {
  Serial.begin(115200);
  Wire.begin(SDA_PIN, SCL_PIN);
  Serial.println("I2C scanner starting...");

  int found = 0;
  for (byte address = 1; address < 127; address++) {
    Wire.beginTransmission(address);
    // endTransmission() returns 0 on ACK (device present), documented
    // by the Wire library as the standard presence-detection method.
    byte error = Wire.endTransmission();
    if (error == 0) {
      Serial.printf("Device found at 0x%02X\n", address);
      found++;
    }
  }
  Serial.printf("Scan complete, %d device(s) found\n", found);
}

void loop() {}
```

## Reading a BME280 (temperature/humidity/pressure) over I2C

The BME280 is a common combined environmental sensor, typically at I2C
address `0x76` or `0x77`. Adafruit's `Adafruit_BME280` library documents
a `begin(address)` call and simple accessor methods:

```cpp
// i2c-bme280-read.ino
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BME280.h>

Adafruit_BME280 bme;

void setup() {
  Serial.begin(115200);
  Wire.begin(); // default pins for the board

  // begin() returns false if it can't find/verify the chip ID over I2C --
  // documented failure mode for a wrong address or bad wiring.
  if (!bme.begin(0x76)) {
    Serial.println("Could not find BME280 sensor, check wiring/address!");
    while (true) delay(1000);
  }
}

void loop() {
  Serial.printf("Temp: %.2fC  Humidity: %.1f%%  Pressure: %.1fhPa\n",
                bme.readTemperature(),
                bme.readHumidity(),
                bme.readPressure() / 100.0F); // library returns Pa; hPa = Pa/100
  delay(2000);
}
```

## Driving an SSD1306 OLED display over I2C

Small 128x64 OLED displays (SSD1306 driver) are another common I2C
peripheral, driven by Adafruit's `Adafruit_SSD1306` + `Adafruit_GFX`
libraries:

```cpp
// i2c-oled-display.ino
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

void setup() {
  Serial.begin(115200);
  Wire.begin();

  // 0x3C is the documented default I2C address for most common
  // SSD1306 128x64 breakout modules.
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 not found");
    while (true) delay(1000);
  }

  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Hello IoT");
  display.display(); // buffered -- nothing shows until display() is called
}

void loop() {}
```

## Combining two I2C devices on one bus

Since the bus is shared, both the BME280 (`0x76`) and SSD1306 (`0x3C`)
can coexist on the same `SDA`/`SCL` wires as long as their addresses
don't collide — the scanner sketch above is exactly how you'd confirm
that before writing combined code.

## Exercise

1. Write the scanner sketch and reason through what addresses you'd
   expect to see for a BME280 + SSD1306 combo (`0x76` and `0x3C`).
2. Write the BME280 read sketch and explain what a `begin()` failure
   (returning `false`) would mean in practice.
3. Write the OLED sketch and explain why `display.display()` is a
   separate call from drawing/printing commands.
4. Combine both into one sketch that shows the BME280's live readings on
   the OLED screen, refreshed every 2 seconds.
