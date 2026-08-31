# Designing a Production IoT Device

!!! note "Not flashed to hardware"
    Reasoned through against documented ESP8266/ESP32 hardware
    reference specs (datasheets, Arduino-core boot behavior) rather than
    a physical build in this environment.

## From "sketch that works on my desk" to "product"

Everything in Levels 1-3 assumed a dev board, USB power, and you
watching the Serial Monitor. A production device removes all three:
it's a custom PCB, powered however the enclosure allows, and no one is
watching it boot. This module is the checklist that bridges the gap.

## 1. Pick the right module for the product, not the prototype

| Concern | ESP8266 | ESP32 |
|---|---|---|
| GPIO count | Limited (~9 usable) | Generous (~25+ usable) |
| Peripherals | I2C, SPI, 1x ADC | I2C, SPI, multiple ADC, BLE, CAN (TWAI), touch |
| RAM headroom | ~50KB free typical | ~300KB+ free typical |
| Power modes | Deep sleep only | Deep/light/modem sleep, per-peripheral gating |
| Cost | Lower | Higher |

A production BOM decision should be justified by the product's actual
GPIO/peripheral/RAM needs, not by "what I prototyped on."

## 2. Boot-time invariants a production sketch must handle

A dev-board sketch usually assumes: USB is present, a human resets it
on failure, and Serial exists to debug. None hold in the field.

```cpp
// production-boot.ino
#include <esp_task_wdt.h>

void setup() {
  // Serial may or may not be attached in the field (no USB host) —
  // never block waiting on it, unlike some dev-board examples do.
  Serial.begin(115200);

  esp_task_wdt_init(8, true); // watchdog from first instruction, not "eventually"
  esp_task_wdt_add(NULL);

  if (!bringUpStorage()) {
    // A device that can't mount its config filesystem still needs a
    // defined behavior — don't silently continue into undefined state.
    enterSafeMode();
    return;
  }

  loadConfig();
  connectWithBoundedRetry(); // never an unbounded while(true) retry loop
}

bool bringUpStorage() { return true; /* LittleFS.begin(true) etc. */ }
void loadConfig() {}
void connectWithBoundedRetry() {}
void enterSafeMode() {
  // Minimal mode: keep the watchdog fed, blink an error pattern,
  // stay reachable for recovery (e.g. captive portal) instead of
  // looping forever on a failed subsystem.
}

void loop() { esp_task_wdt_reset(); }
```

## 3. Power input realities

A prototype runs off a USB cable that never sags. A product might run
off a wall adapter shared with other loads, a battery under load, or a
long cable with voltage drop. Two documented, code-relevant
consequences:

- **Brownout resets**: both cores document a brownout detector that
  resets the chip below a voltage threshold (ESP32: configurable via
  `esp_bootloader`/`board_init`; ESP8266: a fixed hardware BOD). Expect
  a "why did it reboot" reset-reason of "Brownout" under marginal
  power, and design the enclosure's power supply with headroom, not
  just firmware.
- **Inrush at boot**: WiFi TX draws current spikes (documented up to
  ~300-400mA on both chips during transmission) that a thin USB cable
  or weak regulator can't supply, causing resets that look like
  software bugs but are power design bugs.

## 4. Enclosure and antenna considerations that affect firmware

- A metal enclosure attenuates the onboard PCB antenna; if the product
  design requires one, code should degrade gracefully (retry with
  backoff, surface a "poor signal" status) rather than assume WiFi
  always succeeds quickly, since RSSI will be worse than on the bench.
- Thermal buildup in a sealed enclosure can push the chip toward its
  documented operating temperature ceiling (typically 85°C on both
  chips) under sustained WiFi TX + sensor polling — firmware that
  reduces duty cycle if internal temperature (if measurable) trends
  high is a legitimate mitigation.

## 5. The production readiness checklist

1. Every network call has a timeout and a bounded retry (Level 3 wired
   this in per-subsystem — production means auditing *all* of them).
2. The watchdog covers 100% of code paths, not just `loop()`'s happy
   path (Level 3.04).
3. First boot with no config lands in provisioning mode, not a crash
   loop (Level 3.07).
4. OTA updates self-validate and can roll back (Level 3.08).
5. Diagnostics ship off-device so a bug is debuggable without physical
   access (Level 3.09).
6. Reset reason and boot count are logged, so a brownout-reset loop is
   visible in aggregate across the fleet, not just per-device.

## Exercise

1. Take the `production-boot.ino` skeleton and implement
   `connectWithBoundedRetry()` with exponential backoff, capped at 5
   attempts before falling back to `enterSafeMode()`.
2. Explain, using the documented brownout-reset behavior, why a device
   that reboots repeatedly under load might be a power-supply problem
   even when the firmware itself has no bugs.
3. Propose two firmware-level mitigations for reduced WiFi range caused
   by a metal enclosure, without changing the antenna hardware.
4. Design a boot counter stored in LittleFS/NVS that trips
   `enterSafeMode()` after 3 crash-reboots in a row, and resets to 0
   after 10 minutes of stable uptime.
