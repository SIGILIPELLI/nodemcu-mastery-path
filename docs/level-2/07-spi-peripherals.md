# 07 · SPI Peripherals

!!! note "Not flashed to hardware"
    Reasoned through against the Arduino core's documented `SPI` library
    API (`SPI.begin()`, `SPI.beginTransaction()`, `SPISettings`) shared by
    ESP8266 and ESP32, and Adafruit's documented `Adafruit_ST7735`/SD
    library APIs as representative SPI peripheral drivers. Not compiled
    or flashed to physical hardware in this environment.

## I2C vs. SPI

Where I2C (module 06) shares two wires among many addressed devices,
**SPI** ("Serial Peripheral Interface") uses more wires but moves data
much faster and doesn't need addressing at all — each device gets its
own dedicated **CS/SS** (Chip Select) line, while `MOSI`, `MISO`, and
`SCK` are shared. SPI is the standard choice for anything that needs
higher throughput than I2C comfortably provides: SD cards, TFT/color
displays, and fast ADCs.

On NodeMCU-style ESP8266 boards, hardware SPI pins are fixed:
`D7`=MOSI, `D6`=MISO, `D5`=SCK, and you choose any free GPIO as CS. ESP32
defaults to GPIO23=MOSI, GPIO19=MISO, GPIO18=SCK (VSPI), also with a
free-choice CS pin.

## Basic SPI transaction pattern

```cpp
// spi-basic-transaction.ino
#include <SPI.h>

const int CS_PIN = D8; // ESP8266; pick any free GPIO on ESP32

void setup() {
  Serial.begin(115200);
  pinMode(CS_PIN, OUTPUT);
  digitalWrite(CS_PIN, HIGH); // deselected by default (active-low CS)
  SPI.begin();
}

byte readRegister(byte reg) {
  // SPISettings documents clock speed, bit order, and SPI mode --
  // wrapping each transaction lets multiple SPI devices at different
  // speeds share the same bus safely.
  SPI.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
  digitalWrite(CS_PIN, LOW); // select this device
  SPI.transfer(reg);          // send the register address to read
  byte value = SPI.transfer(0x00); // clock out a dummy byte, read the reply
  digitalWrite(CS_PIN, HIGH); // deselect
  SPI.endTransaction();
  return value;
}

void loop() {
  byte value = readRegister(0x0F);
  Serial.printf("Register 0x0F: 0x%02X\n", value);
  delay(1000);
}
```

The exact register protocol (what byte to send, what comes back) is
chip-specific and documented per datasheet — the pattern above (select,
transfer, deselect, wrapped in a transaction) is the universal shape
almost every SPI driver library follows internally.

## Driving an ST7735 color TFT display

```cpp
// spi-tft-display.ino
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7735.h>

#define TFT_CS   D8
#define TFT_DC   D3  // data/command select line, SPI displays need this too
#define TFT_RST  D4

Adafruit_ST7735 tft(TFT_CS, TFT_DC, TFT_RST);

void setup() {
  Serial.begin(115200);
  // INITR_BLACKTAB is documented as the common default for most
  // 1.8" ST7735 breakout modules; other tab colors exist for other
  // manufacturing runs of the same chip.
  tft.initR(INITR_BLACKTAB);
  tft.fillScreen(ST77XX_BLACK);
  tft.setTextColor(ST77XX_WHITE);
  tft.setTextSize(2);
  tft.setCursor(0, 0);
  tft.println("Hello SPI!");
}

void loop() {}
```

## Reading/writing an SD card over SPI

```cpp
// spi-sd-card.ino
#include <SPI.h>
#include <SD.h>

const int SD_CS_PIN = D8;

void setup() {
  Serial.begin(115200);
  // SD.begin(csPin) documents returning false on a missing/unformatted
  // card or wrong CS pin -- always guard on it before using the card.
  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("SD card init failed!");
    return;
  }

  File logFile = SD.open("/log.txt", FILE_WRITE); // FILE_WRITE appends
  if (logFile) {
    logFile.println("Sensor node booted");
    logFile.close(); // must close to flush -- data can be lost otherwise
    Serial.println("Wrote to log.txt");
  } else {
    Serial.println("Failed to open log.txt");
  }
}

void loop() {}
```

## Sharing an SPI bus between two devices

Both the TFT and SD card examples can share `MOSI`/`MISO`/`SCK` since
each uses its own CS pin — the key rule (documented by the `SPI` library
and followed above) is that only one device's CS should ever be `LOW`
at a time, which `beginTransaction()`/`endTransaction()` pairs make easy
to guarantee.

## Exercise

1. Write the basic register-read sketch and explain what each of
   `SPISettings`'s three arguments controls.
2. Write the TFT sketch and explain why `TFT_DC` is needed for SPI
   displays but not for something like an SD card.
3. Write the SD card sketch and explain the documented failure mode when
   `SD.begin()` returns false.
4. Sketch (in comments) how you'd wire a TFT and SD card on the same SPI
   bus with two separate CS pins, and confirm both `beginTransaction()`
   calls never overlap.
