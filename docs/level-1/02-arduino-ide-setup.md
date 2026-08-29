# 02 · Setting Up the Arduino IDE for ESP Boards

!!! note "Not flashed to hardware"
    The steps below are reasoned through against the documented Arduino
    IDE / board-manager behavior and the official ESP8266 and ESP32
    Arduino-core installation instructions, but have not been executed in
    this environment. Screen layouts change slightly between IDE versions
    — use the menu names as a guide, not pixel-exact positions.

## 1. Install the Arduino IDE

Download and install the current **Arduino IDE 2.x** from
[arduino.cc/en/software](https://www.arduino.cc/en/software) for your OS.
IDE 2.x has a built-in Boards Manager and Library Manager, both of which
this module uses. (Everything here also works on the legacy 1.8.x IDE, with
slightly different menu wording.)

## 2. Add the ESP8266 and ESP32 board manager URLs

Both chip families are third-party cores, not bundled with the IDE by
default, so you point the IDE at their package index URLs first.

1. Open **File → Preferences** (macOS: **Arduino IDE → Settings**).
2. Find the field **Additional boards manager URLs**.
3. Paste both URLs, comma-separated, into that field:

```
https://arduino.esp8266.com/stable/package_esp8266com_index.json,https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

4. Click **OK**.

!!! tip "One board family only?"
    If you only own an ESP8266 board, paste only the first URL; if you only
    own an ESP32 board, paste only the second. Having both installed is
    harmless and lets you switch between board types on the same machine.

## 3. Install the board packages

1. Open **Tools → Board → Boards Manager…**.
2. Search `esp8266` and install **"esp8266 by ESP8266 Community"**.
3. Search `esp32` and install **"esp32 by Espressif Systems"**.
4. Wait for both downloads to finish — the ESP32 package in particular is
   large (it bundles a full Xtensa/RISC-V toolchain) and can take several
   minutes on a slow connection.

## 4. Install USB-to-serial drivers

Almost every NodeMCU/ESP32 dev board uses one of two USB-to-serial bridge
chips, and your OS needs a driver for whichever one is on your board:

| Chip | Common on | Driver |
|---|---|---|
| CP2102 / CP2104 | NodeMCU v2/v3, many ESP32 DevKitC boards | Silicon Labs CP210x VCP driver |
| CH340 / CH340G | Cheaper NodeMCU clones, many Wemos boards | CH340 driver (WCH) |

Install the driver that matches the chip printed near the USB connector on
your board. On modern macOS and most Linux distributions, CP210x drivers
are frequently already included in the OS; CH340 drivers usually need a
manual install on macOS and some Linux distros. Windows typically needs a
manual driver install for either chip on older Windows 10 builds.

**How to tell if the driver worked:** plug the board in, then check:

- **Windows** — Device Manager should show a new **COM port** (e.g. `COM5`)
  under "Ports (COM & LPT)", not an "Unknown device" with a yellow warning.
- **macOS** — a new device should appear under `/dev/cu.usbserial-*` (CH340)
  or `/dev/cu.SLAB_USBtoUART` (CP210x). Check with `ls /dev/cu.*` in a
  terminal.
- **Linux** — a new device should appear as `/dev/ttyUSB0` (or similar).
  Check with `ls /dev/ttyUSB*`. You may also need to add your user to the
  `dialout` group (`sudo usermod -a -G dialout $USER`, then log out/in)
  before the IDE can open the port without `sudo`.

## 5. Select your board and port

1. Plug in your board via a **data-capable** USB cable (see the warning in
   Module 01 — a charge-only cable will never show a port).
2. Open **Tools → Board** and pick the entry matching your hardware:
   - ESP8266 NodeMCU board → **Tools → Board → ESP8266 Boards → NodeMCU
     1.0 (ESP-12E Module)**
   - ESP32 DevKitC-style board → **Tools → Board → ESP32 Arduino →
     ESP32 Dev Module**
3. Open **Tools → Port** and select the COM/`/dev/tty*` port that appeared
   when you plugged the board in.
4. Leave other settings (Upload Speed, Flash Size, etc.) at their defaults
   for now — they matter more once you're doing OTA updates or using
   SPIFFS/LittleFS in later levels.

## 6. Verify with a blank sketch compile

Before wiring anything, confirm the toolchain itself works:

1. **File → New Sketch** to get an empty `setup()`/`loop()` template.
2. Click the checkmark **Verify/Compile** button (not Upload yet).
3. Watch the bottom console. A successful compile ends with something like
   `Sketch uses XXXXX bytes (NN%) of program storage space.` If instead you
   see errors referencing missing board packages or toolchain files, revisit
   step 3 and make sure the correct board package finished installing.

Only once compile succeeds should you try an actual **Upload** — which is
exactly what the next module does with a real, visible-result sketch: Blink.

## Exercise

1. Install both board packages (or just the one matching your hardware) via
   Boards Manager as described above.
2. Install the correct USB-to-serial driver for your board's bridge chip.
3. Select the correct board entry and port under the **Tools** menu.
4. Create a new blank sketch and **Verify/Compile** it successfully — do
   not upload yet. Note down the reported program storage size; you'll
   compare it against the Blink sketch's size in the next module to get a
   feel for how much flash a "do-nothing" sketch already uses.
