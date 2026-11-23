---
title: "NodeMCU/IoT Interview Questions and Answers (179 Questions)"
description: "179 NodeMCU/IoT interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the NodeMCU/IoT Mastery Path."
---

# NodeMCU/IoT Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free NodeMCU/IoT Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is NodeMCU (ESP8266 vs ESP32)](level-1/01-what-is-nodemcu.md)

**Q1.** What is NodeMCU?

**Answer:** An open-source development board/firmware built around the ESP8266 (and ESP32 variants) with Wi-Fi

Cheap IoT prototyping.

**Q2.** What does the ESP8266 add over a basic MCU?

**Answer:** Built-in 2.4 GHz Wi-Fi with a TCP/IP stack

Connectivity.

**Q3.** What voltage do NodeMCU GPIO pins use?

**Answer:** 3.3 V (not 5 V tolerant)

Level shifters for 5 V.

**Q4.** What's the board's USB chip for?

**Answer:** USB-serial to program and monitor the module

CP2102/CH340.

### [02 · Setting Up the Arduino IDE for ESP Boards](level-1/02-arduino-ide-setup.md)

**Q5.** How add NodeMCU support to the Arduino IDE?

**Answer:** Install the ESP8266 (or ESP32) board package via Boards Manager

Board URL.

**Q6.** What must be selected before uploading?

**Answer:** The correct board and serial port

Tools menu.

**Q7.** What if the port doesn't appear?

**Answer:** Install the USB-serial driver and use a data-capable cable

Common issue.

**Q8.** What baud rate is typical for upload?

**Answer:** 115200 (configurable)

Upload speed.

### [03 · First Blink Sketch](level-1/03-first-blink-sketch.md)

**Q9.** What does `setup()` do?

**Answer:** Runs once at startup for initialization

pinMode etc.

**Q10.** What does `loop()` do?

**Answer:** Runs repeatedly forever after setup

Main logic.

**Q11.** Why is the onboard LED often active-low on NodeMCU?

**Answer:** It's wired to 3.3 V so LOW turns it on

digitalWrite(LED_BUILTIN, LOW).

**Q12.** Why avoid long delay() calls?

**Answer:** They block everything, including Wi-Fi background work

Use millis().

### [04 · GPIO Basics: Digital Read &amp; Write](level-1/04-gpio-basics.md)

**Q13.** What does pinMode(pin, INPUT_PULLUP) do?

**Answer:** Enables an internal pull-up so an unconnected input reads HIGH

Buttons to ground.

**Q14.** Which pins are risky at boot on the ESP8266?

**Answer:** Strapping pins like GPIO0, GPIO2 and GPIO15

Wrong levels prevent boot.

**Q15.** What current can a GPIO source?

**Answer:** Only a few mA; drive loads through a transistor

Protect the chip.

**Q16.** What's D1 vs GPIO5?

**Answer:** D1 is the board silkscreen label for GPIO5

Mapping tables.

### [05 · Analog Input (ADC)](level-1/05-analog-input-adc.md)

**Q17.** How many ADC channels does the ESP8266 have?

**Answer:** One (A0) with a 0–1 V range on the chip

Boards add a divider.

**Q18.** What does this print (10-bit ADC value 512 mapped to 0..100)?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
long map(long x, long in_min, long in_max, long out_min, long out_max){ return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min; }
int main(void){ printf("%ld", map(512, 0, 1023, 0, 100)); }
```

**Answer:** 50

Arduino map() uses integer arithmetic. Verified by compiling and running.

**Q19.** Why average several ADC samples?

**Answer:** The ADC is noisy, especially with Wi-Fi active

Smoothing.

**Q20.** What's the NodeMCU A0 input range?

**Answer:** About 0–3.3 V (a divider on the board)

Check the schematic.

### [06 · PWM Basics](level-1/06-pwm-basics.md)

**Q21.** What does analogWrite do on the ESP8266?

**Answer:** Software PWM with 10-bit resolution (0–1023)

Default range.

**Q22.** What duty cycle (percent) is analogWrite value 256 with a 0..1023 range?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
int main(void){ printf("%.1f", 256 * 100.0 / 1023); }
```

**Answer:** 25.0

Value / range. Verified by compiling and running.

**Q23.** What does PWM frequency affect?

**Answer:** Flicker and motor/LED behavior

analogWriteFreq.

**Q24.** Why can't PWM drive a motor directly from a pin?

**Answer:** Pins source too little current; use a driver/transistor

Flyback diode.

### [07 · Serial Monitor Debugging](level-1/07-serial-monitor-debugging.md)

**Q25.** What does Serial.begin(115200) do?

**Answer:** Opens the serial port at that baud rate

Match the monitor.

**Q26.** What's garbage text in the monitor usually?

**Answer:** A baud rate mismatch (bootloader prints at 74880 on ESP8266)

Set matching speed.

**Q27.** What's a good debug practice?

**Answer:** Print state and values with labels, remove noisy logs later

Readable logs.

**Q28.** What can excess Serial printing do?

**Answer:** Slow the loop and affect timing

Use sparingly.

### [08 · Connecting to WiFi](level-1/08-connecting-to-wifi.md)

**Q29.** Which library connects the ESP8266 to Wi-Fi?

**Answer:** ESP8266WiFi (WiFi.h on ESP32)

WiFi.begin(ssid, pw).

**Q30.** How check connection?

**Answer:** Poll WiFi.status() until WL_CONNECTED with a timeout

Don't loop forever.

**Q31.** What should firmware do on disconnect?

**Answer:** Reconnect with backoff rather than hang

Robustness.

**Q32.** Where store credentials?

**Answer:** Not hard-coded in source shared publicly; use config/provisioning

Security.

### [09 · Reading a Sensor (DHT11 Temperature)](level-1/09-reading-a-sensor-dht11.md)

**Q33.** What does a DHT11 measure?

**Answer:** Temperature and humidity (low accuracy)

Cheap sensor.

**Q34.** How often can a DHT11 be read?

**Answer:** About once per second at most

Datasheet limit.

**Q35.** What does a NaN reading mean?

**Answer:** The read failed; handle it instead of using the value

isnan().

**Q36.** What pull-up does the DHT data line need?

**Answer:** A resistor (often built into the module board)

Reliable reads.

### [10 · Capstone — WiFi Sensor Node &amp; Web Server](level-1/10-capstone-wifi-sensor-node.md)

**Q37.** What does the Wi-Fi sensor node combine?

**Answer:** Sensor reading, Wi-Fi connection and reporting over the network

Level 1 capstone.

**Q38.** How structure the loop?

**Answer:** Non-blocking timers (millis) for sampling and reporting

Responsive.

**Q39.** How handle failed uploads?

**Answer:** Retry later and keep the latest data

Resilience.

**Q40.** What proves it works?

**Answer:** Serial logs and receiving data at the server

End to end.

## Level 2 · Intermediate

### [01 · MQTT Basics for IoT Messaging](level-2/01-mqtt-basics.md)

**Q41.** What is MQTT?

**Answer:** A lightweight publish/subscribe protocol via a broker

IoT messaging.

**Q42.** What's a topic?

**Answer:** A hierarchical string that routes messages (home/kitchen/temp)

Wildcards + #.

**Q43.** What do QoS 0/1/2 mean?

**Answer:** At most once, at least once and exactly once

Overhead rises.

**Q44.** What is a Last Will message?

**Answer:** Published by the broker if a client disconnects unexpectedly

Presence.

### [02 · Connecting to a Cloud IoT Platform](level-2/02-cloud-iot-platform.md)

**Q45.** What do cloud IoT platforms provide?

**Answer:** Device connectivity, dashboards, storage and rules

ThingSpeak, AWS IoT.

**Q46.** How authenticate devices?

**Answer:** Per-device credentials or certificates

Revocable.

**Q47.** Why rate limit uploads?

**Answer:** Platforms throttle and battery/data costs rise

Respect limits.

**Q48.** What data format is common?

**Answer:** JSON payloads

Interoperable.

### [03 · Deep Sleep &amp; Power Management](level-2/03-deep-sleep-power-management.md)

**Q49.** What does deep sleep do?

**Answer:** Powers down the CPU and Wi-Fi, waking on a timer or pin (microamps)

ESP.deepSleep.

**Q50.** What must GPIO16 connect to for timer wake on ESP8266?

**Answer:** RST (so it can reset the chip)

Wiring.

**Q51.** Wake 3 s at 80 mA every 5 minutes, sleeping at 0.02 mA. Average current in mA?

```python
print(round((3*80+297*0.02)/300,3))
```

**Answer:** 0.82

Duty-cycle average. Verified by running it.

**Q52.** What survives deep sleep?

**Answer:** RTC memory (a little) and flash, but not RAM

Store state.

### [04 · OTA (Over-the-Air) Updates](level-2/04-ota-updates.md)

**Q53.** What does OTA update do?

**Answer:** Updates firmware over Wi-Fi without a cable

ArduinoOTA.

**Q54.** What must OTA protect against?

**Answer:** Interrupted or malicious updates (verify and authenticate)

Passwords/signing.

**Q55.** Why keep a fallback?

**Answer:** A failed update shouldn't brick a remote device

Dual partitions/rollback.

**Q56.** What does the OTA need free?

**Answer:** Flash space for the new image beside the running one

Partition scheme.

### [05 · Interrupts &amp; Debouncing](level-2/05-interrupts-debouncing.md)

**Q57.** Why debounce a button?

**Answer:** Mechanical contacts bounce, creating many edges

Time filter.

**Q58.** What should an ISR do?

**Answer:** Minimal work: set a flag and return (IRAM_ATTR on ESP)

Keep short.

**Q59.** What qualifier do shared ISR variables need?

**Answer:** volatile

Avoid caching.

**Q60.** Why not call Serial or delay in an ISR?

**Answer:** They can block or crash; defer to the main loop

Safety.

### [06 · I2C Peripherals](level-2/06-i2c-peripherals.md)

**Q61.** What is I2C?

**Answer:** A two-wire bus (SDA, SCL) connecting many devices by address

Wire library.

**Q62.** What does an I2C scanner find?

**Answer:** Device addresses on the bus

Debug wiring.

**Q63.** Why pull-up resistors on I2C?

**Answer:** The bus is open-drain so devices only pull low

4.7 kΩ.

**Q64.** Default I2C pins on NodeMCU?

**Answer:** D2 (SDA) and D1 (SCL)

GPIO4/5.

### [07 · SPI Peripherals](level-2/07-spi-peripherals.md)

**Q65.** What does SPI use?

**Answer:** Clock, MOSI, MISO and a chip-select per device

Faster than I2C.

**Q66.** What defines an SPI mode?

**Answer:** Clock polarity and phase (CPOL/CPHA)

Match the device.

**Q67.** When prefer SPI over I2C?

**Answer:** Higher speed needs like displays and SD cards

Throughput.

**Q68.** What's CS for?

**Answer:** Selecting which device is active

One per device.

### [08 · JSON Data &amp; Config Storage](level-2/08-json-data-config-storage.md)

**Q69.** Which library handles JSON on Arduino?

**Answer:** ArduinoJson

Fixed-size documents.

**Q70.** Why size JSON documents carefully?

**Answer:** RAM is small and overflows fail silently or crash

StaticJsonDocument.

**Q71.** Where store config persistently?

**Answer:** LittleFS files or EEPROM/Preferences

Survives reboot.

**Q72.** What should config loading do on corrupt data?

**Answer:** Fall back to defaults safely

Robust.

### [09 · Multi-Device Networking Patterns](level-2/09-multi-device-networking.md)

**Q73.** How do multiple devices find each other?

**Answer:** Via a broker/server or discovery (mDNS, ESP-NOW)

Choose by topology.

**Q74.** What does mDNS provide?

**Answer:** Local hostnames like device.local without a DNS server

Zero-config.

**Q75.** What's ESP-NOW?

**Answer:** Connectionless peer-to-peer messaging between ESP devices

Low power.

**Q76.** Why unique client IDs for MQTT?

**Answer:** Duplicate IDs kick each other off the broker

Use chip ID.

### [10 · Capstone — MQTT Sensor-to-Cloud Bridge](level-2/10-capstone-mqtt-sensor-bridge.md)

**Q77.** What does the MQTT sensor bridge combine?

**Answer:** Sensors, MQTT publishing, retained status and reconnection logic

Level 2 capstone.

**Q78.** What's a retained message good for?

**Answer:** Giving new subscribers the last known state immediately

Status topics.

**Q79.** How handle broker downtime?

**Answer:** Reconnect with backoff and buffer recent data

Resilience.

**Q80.** How verify end to end?

**Answer:** Subscribe with a client and check payload and timing

mosquitto_sub.

## Level 3 · Advanced

### [01 · Building a REST API on the ESP32](level-3/01-rest-api-esp32.md)

**Q81.** How serve a REST API on the ESP?

**Answer:** A web server library with route handlers returning JSON

ESP8266WebServer.

**Q82.** Which method reads data and which changes it?

**Answer:** GET reads; POST/PUT change state

Semantics.

**Q83.** What status for invalid input?

**Answer:** 400 Bad Request

Validate.

**Q84.** Why add authentication?

**Answer:** Anyone on the network could control the device

Tokens.

### [02 · Securing IoT Communication (TLS Basics)](level-3/02-securing-iot-tls-basics.md)

**Q85.** What does TLS provide?

**Answer:** Encryption and server authentication over the connection

HTTPS/MQTTS.

**Q86.** What's a challenge on small MCUs?

**Answer:** TLS needs RAM, CPU and certificate storage

Size buffers.

**Q87.** Why verify the server certificate?

**Answer:** Prevent man-in-the-middle attacks

Pin CA/fingerprint.

**Q88.** What's wrong with setInsecure()?

**Answer:** It skips verification and is only for testing

Don't ship it.

### [03 · Multi-Sensor Data Aggregation &amp; Buffering](level-3/03-multi-sensor-aggregation-buffering.md)

**Q89.** What's aggregation buffering?

**Answer:** Collecting readings and sending batches to save power and bandwidth

Efficiency.

**Q90.** Why ring buffers?

**Answer:** Fixed memory with overwrite of the oldest data

Bounded RAM.

**Q91.** What happens if the network is down?

**Answer:** Buffer locally up to a limit then drop oldest or compress

Policy.

**Q92.** What timestamps do readings need?

**Answer:** Real time via NTP or a relative counter plus sync

Ordering.

### [04 · Watchdog Timers &amp; Reliability](level-3/04-watchdog-timers-reliability.md)

**Q93.** What does a watchdog do?

**Answer:** Resets the device if the firmware stops feeding it

Recovery.

**Q94.** What's a hazard of long blocking calls?

**Answer:** They can trigger the watchdog reset

yield() / feed.

**Q95.** Where feed the watchdog?

**Answer:** In the main loop only when the system is healthy

No blind feeding.

**Q96.** What to do after a watchdog reset?

**Answer:** Log the reason and recover to a safe state

Diagnostics.

### [Task Scheduling on a Single Core](level-3/05-task-scheduling-single-core.md)

**Q97.** How schedule tasks on a single-core MCU?

**Answer:** Cooperative scheduling with millis() timers or a small scheduler library

Non-blocking.

**Q98.** What breaks cooperative scheduling?

**Answer:** A task that blocks for a long time

Keep tasks short.

**Q99.** Why avoid delay() in schedulers?

**Answer:** It stalls all other tasks

State machines.

**Q100.** What's a state machine for?

**Answer:** Breaking long operations into steps across loop iterations

Responsive.

### [Local Data Logging (SPIFFS/LittleFS)](level-3/06-local-data-logging.md)

**Q101.** Where can a device log data locally?

**Answer:** LittleFS/SD card files (with rotation)

Persistence.

**Q102.** Why rotate logs?

**Answer:** Flash is small and wears; unbounded logs fill storage

Bounded.

**Q103.** What limits flash writes?

**Answer:** Limited erase cycles, so batch and avoid frequent writes

Wear.

**Q104.** What if power drops during a write?

**Answer:** The file may corrupt; use atomic rename/journaling patterns

Resilience.

### [Captive Portal &amp; WiFi Provisioning](level-3/07-captive-portal-provisioning.md)

**Q105.** What's a captive portal for?

**Answer:** Letting users enter Wi-Fi credentials by connecting to the device's own AP

Provisioning.

**Q106.** What does the device run during provisioning?

**Answer:** A soft AP with a small web server and DNS redirect

WiFiManager.

**Q107.** Where store the entered credentials?

**Answer:** In flash (LittleFS/Preferences), then reboot into station mode

Persist.

**Q108.** Why a timeout?

**Answer:** Return to the portal or retry if connection fails

Recovery.

### [Firmware Versioning &amp; Rollback](level-3/08-firmware-versioning-rollback.md)

**Q109.** What should firmware report?

**Answer:** Its version and build info

Support.

**Q110.** What's rollback?

**Answer:** Reverting to the previous firmware if the new one fails health checks

Safe updates.

**Q111.** How decide an update is good?

**Answer:** The new firmware confirms it booted and connected

Self-test.

**Q112.** Why semantic versioning?

**Answer:** Communicates compatibility of releases

MAJOR.MINOR.PATCH.

### [Diagnostics &amp; Remote Logging](level-3/09-diagnostics-remote-logging.md)

**Q113.** What remote diagnostics help?

**Answer:** Uptime, RSSI, free heap, reset reason and error counters

Health.

**Q114.** How send logs remotely?

**Answer:** MQTT topics or HTTP endpoints with rate limits

Bandwidth.

**Q115.** Why limit log volume?

**Answer:** Battery, bandwidth and storage

Levels.

**Q116.** What does free heap trend reveal?

**Answer:** Memory leaks over time

Fix leaks.

### [Capstone — Secure Multi-Sensor IoT Gateway](level-3/10-capstone-iot-gateway.md)

**Q117.** What does the IoT gateway combine?

**Answer:** Sensors/nodes aggregation, buffering, secure uplink and diagnostics

Level 3 capstone.

**Q118.** How prove reliability?

**Answer:** Soak tests with network loss and power cycles

Evidence.

**Q119.** What should it expose?

**Answer:** A status endpoint and logs

Operations.

**Q120.** What guards updates?

**Answer:** Signed firmware with rollback

Safety.

## Level 4 · Master

### [Designing a Production IoT Device](level-4/01-production-iot-device-design.md)

**Q121.** What's production IoT design different from prototyping?

**Answer:** Reliability, security, updates, power and manufacturability are designed in

Product thinking.

**Q122.** What are key requirements?

**Answer:** Power budget, connectivity, security and lifecycle support

Constraints.

**Q123.** Why design for failure?

**Answer:** Devices run unattended for years in uncontrolled conditions

Fault tolerance.

**Q124.** What should be defined early?

**Answer:** The lifecycle: provisioning, updates, monitoring and end of life

Plan.

### [Provisioning at Scale](level-4/02-provisioning-at-scale.md)

**Q125.** What's provisioning at scale?

**Answer:** Securely giving each device a unique identity and credentials during manufacturing

Factory flow.

**Q126.** Why unique per-device credentials?

**Answer:** A leaked shared secret compromises the whole fleet

Revocation.

**Q127.** What does zero-touch onboarding mean?

**Answer:** Devices self-register securely on first connection

No manual setup.

**Q128.** Where store keys?

**Answer:** In secure storage/eFuse or a secure element

Protect.

### [Fleet Management Concepts](level-4/03-fleet-management-concepts.md)

**Q129.** What does fleet management provide?

**Answer:** Monitoring, configuration, updates and grouping of many devices

Scale.

**Q130.** What's a device twin/shadow?

**Answer:** A cloud record of desired and reported device state

Sync.

**Q131.** Why group devices?

**Answer:** Staged rollouts and targeted configuration

Cohorts.

**Q132.** What telemetry matters?

**Answer:** Version, health, connectivity and battery

Visibility.

### [OTA Update Strategy at Scale](level-4/04-ota-strategy-at-scale.md)

**Q133.** What makes OTA safe at scale?

**Answer:** Staged rollouts, signed images, health checks and automatic rollback

Canary.

**Q134.** Why stagger updates?

**Answer:** Limit blast radius and server load

Waves.

**Q135.** What if devices are offline for months?

**Answer:** Handle version skew with resumable and compatible updates

Compatibility.

**Q136.** How reduce update size?

**Answer:** Delta/differential updates and compression

Bandwidth.

### [Power Budget Design for Battery Devices](level-4/05-power-budget-design.md)

**Q137.** What's a power budget?

**Answer:** Estimating average current from duty cycles of each state

Battery life.

**Q138.** Battery 2000 mAh, average draw 0.5 mA: days of life?

```python
print(round(2000/0.5/24,1))
```

**Answer:** 166.7

capacity/current in hours. Verified by running it.

**Q139.** What dominates Wi-Fi device power?

**Answer:** Radio transmit/receive and wake time

Minimize connect time.

**Q140.** How reduce wake time?

**Answer:** Use static IP, fast connect and send small batches

Efficiency.

### [Edge Processing vs. Cloud Offload](level-4/06-edge-vs-cloud-offload.md)

**Q141.** When process data at the edge?

**Answer:** To cut bandwidth/latency or work offline

Filter locally.

**Q142.** When offload to the cloud?

**Answer:** Heavy analytics, aggregation and storage

Capacity.

**Q143.** What's a hybrid approach?

**Answer:** Local filtering/alerts with cloud analytics

Balance.

**Q144.** What does edge processing risk?

**Answer:** Limited compute/memory and harder updates

Trade-offs.

### [Security Hardening for Production IoT](level-4/07-security-hardening.md)

**Q145.** What hardens an IoT device?

**Answer:** Secure boot, signed OTA, TLS, minimal services and disabled debug ports

Defense.

**Q146.** Why change default credentials?

**Answer:** Default passwords are the top IoT attack vector

Unique per device.

**Q147.** What's flash encryption for?

**Answer:** Protecting firmware and secrets from readout

At rest.

**Q148.** What should be disabled in production?

**Answer:** UART/JTAG debug access (or protected)

Reduce attack surface.

### [Certification &amp; Regulatory Basics](level-4/08-certification-regulatory-basics.md)

**Q149.** What do certifications cover?

**Answer:** Radio (FCC/CE), safety (UL/EN) and sometimes security labels

Legal sale.

**Q150.** Why use pre-certified modules?

**Answer:** Reduce radio certification effort and cost

Module certs.

**Q151.** What's EMC testing?

**Answer:** Ensuring the product doesn't emit/interfere beyond limits

Compliance.

**Q152.** What's RoHS?

**Answer:** Restriction of hazardous substances in electronics

Materials.

### [Cost Engineering &amp; BOM Optimization](level-4/09-cost-engineering-bom.md)

**Q153.** What's in a BOM?

**Answer:** All parts with values, footprints, suppliers and costs

Cost engineering.

**Q154.** How reduce BOM cost?

**Answer:** Fewer parts, standard components and volume pricing

Design for cost.

**Q155.** What's DFM?

**Answer:** Design for manufacturing: easy and cheap to build and test

Yield.

**Q156.** Why secondary sources?

**Answer:** Avoid supply-chain risk from a single supplier

Resilience.

### [Capstone — Production IoT Product Design](level-4/10-capstone-production-iot-product.md)

**Q157.** What does the production IoT product capstone combine?

**Answer:** Provisioning, OTA, power budget, security and certification planning

Product.

**Q158.** How prove readiness?

**Answer:** Field trials, soak tests, security review and staged rollout

Evidence.

**Q159.** What documentation helps?

**Answer:** Architecture, runbooks and support procedures

Operate.

**Q160.** What sustains the product?

**Answer:** Updates, monitoring and customer support

Lifecycle.

## Mixed questions across all levels

**Q161.** What voltage do NodeMCU GPIO pins tolerate?

**Answer:** 3.3 V only: use a level shifter or divider for 5 V signals

Protect the pins.

**Q162.** What does Arduino's map() give for a 10-bit reading of 512 mapped to 0..100?

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>
long map(long x, long in_min, long in_max, long out_min, long out_max){ return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min; }
int main(void){ printf("%ld %ld", map(512, 0, 1023, 0, 100), map(1023, 0, 1023, 0, 100)); }
```

**Answer:** 50 100

Integer arithmetic. Verified by compiling and running.

**Q163.** What duty cycle is analogWrite(256) on a 0..1023 scale?

```python
print(round(256*100/1023,1))
```

**Answer:** 25.0

Value divided by range. Verified by running it.

**Q164.** The board boots only when a sensor is unplugged. What's a likely cause?

**Answer:** The sensor loads a strapping pin (GPIO0/2/15) to the wrong level at boot

Check boot-mode pins.

**Q165.** Why does `delay(10000)` in loop() hurt a Wi-Fi device?

**Answer:** It blocks everything including background Wi-Fi/network handling; use millis()

Cooperative design.

**Q166.** Garbage characters appear in the serial monitor. What do you check first?

**Answer:** That the monitor baud rate matches Serial.begin()

Boot ROM messages use 74880.

**Q167.** A DHT11 read returns NaN. What should the code do?

**Answer:** Detect it (isnan), skip or retry and avoid sending bad data

Handle failures.

**Q168.** The device must survive broker and Wi-Fi outages for hours. What design?

**Answer:** Reconnect with backoff and buffer readings in a bounded ring buffer

Resilience.

**Q169.** What does MQTT's Last Will do?

**Answer:** Tells the broker to publish a message if the client disconnects unexpectedly

Presence/offline detection.

**Q170.** Wake 3 s at 80 mA every 5 min, sleeping 297 s at 0.02 mA: average current in mA?

```python
print(round((3*80+297*0.02)/300,3))
```

**Answer:** 0.82

Duty-cycled average. Verified by running it.

**Q171.** For timer wake from deep sleep on an ESP8266, what must be wired?

**Answer:** GPIO16 connected to RST

So the RTC can reset the chip.

**Q172.** An OTA update is interrupted by a power cut. What prevents a brick?

**Answer:** Updating into a spare slot, verifying it and keeping a rollback path

Safe updates.

**Q173.** An ISR increments a counter read by loop(). What must you do?

**Answer:** Declare it volatile, keep the ISR short and avoid Serial/delay inside it

IRAM_ATTR on ESP.

**Q174.** Two sensors on I2C don't respond. What do you do first?

**Answer:** Run an I2C scanner and check pull-ups, addresses and wiring

Debug the bus.

**Q175.** Why validate and size ArduinoJson documents?

**Answer:** RAM is small and oversized or malformed data can overflow or crash

Bounded memory.

**Q176.** setInsecure() works for HTTPS in the lab. Why not ship it?

**Answer:** It skips certificate verification, enabling man-in-the-middle attacks

Pin a CA or fingerprint.

**Q177.** The firmware hangs once a month in the field. What's a standard protection?

**Answer:** A hardware/software watchdog fed only when the system is healthy

Self-recovery.

**Q178.** Every device shares one Wi-Fi/MQTT password in the firmware. What's the risk?

**Answer:** One leak compromises the whole fleet; provision unique per-device credentials

Revocation.

**Q179.** You must roll out firmware to 10,000 devices. What strategy?

**Answer:** Staged (canary) rollout of signed images with health checks and automatic rollback

Limit blast radius.

## Practical coding questions

### A non-blocking scheduler and button debounce

Implement poll() so each task runs when its period has elapsed using a rollover-safe millis comparison (and no drift), and debounce() so the stable button level flips only after three identical consecutive samples. The program starts near the 32-bit millis() rollover.

Starter code:

```
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

/* Non-blocking scheduler driven by a fake millis() (like Arduino). Each task runs every `period` ms. */
typedef struct { const char *name; uint32_t period, next_due; int runs; } Task;
static uint32_t fake_ms = 0;
static uint32_t millis(void) { return fake_ms; }

typedef struct { bool stable; bool last; int count; } Deb;

static void poll(Task *t, int n, char *log) {
    /* run every task whose due time has been reached using the wrap-safe test (int32_t)(millis() - next_due) >= 0:
       increment runs, advance next_due by period (no drift) and append "<name> " to log */
}

static bool debounce(Deb *d, bool sample, int need, bool *changed) {
    /* *changed is true only on the sample where the stable level flips; the stable level changes only after
       `need` identical consecutive samples that differ from the current stable level */
    *changed = false;
    return d->stable;
}

int main(void) {
    Task tasks[] = { {"blink", 500, 500, 0}, {"sensor", 2000, 2000, 0}, {"report", 5000, 5000, 0} };
    char log[256] = "";
    fake_ms = 0xFFFFFFFFu - 700;                       /* start near the 32-bit millis() rollover */
    for (int i = 0; i < 3; i++) tasks[i].next_due = fake_ms + tasks[i].period;
    for (int step = 0; step < 12; step++) { fake_ms += 1000; poll(tasks, 3, log); }
    printf("%s\n", log);
    printf("runs: blink=%d sensor=%d report=%d\n", tasks[0].runs, tasks[1].runs, tasks[2].runs);

    int samples[] = {0,1,0,0,1,1,1,1,0,1,0,0,0,0};
    Deb d = {false, false, 0};
    for (unsigned i = 0; i < sizeof samples / sizeof *samples; i++) {
        bool ch; debounce(&d, samples[i], 3, &ch);
        if (ch) printf("sample %u: %s\n", i, d.stable ? "PRESSED" : "RELEASED");
    }
    return 0;
}
```

Expected output:

```text
blink blink sensor blink blink sensor blink report blink sensor blink blink sensor blink blink sensor report blink blink sensor 
runs: blink=12 sensor=6 report=2
sample 6: PRESSED
sample 12: RELEASED
```

??? success "Show a solution"

    ```
    #include <stdio.h>
    #include <stdint.h>
    #include <stdbool.h>
    #include <string.h>

    /* Non-blocking scheduler driven by a fake millis() (like Arduino). Each task runs every `period` ms. */
    typedef struct { const char *name; uint32_t period, next_due; int runs; } Task;
    static uint32_t fake_ms = 0;
    static uint32_t millis(void) { return fake_ms; }

    static void poll(Task *t, int n, char *log) {
        for (int i = 0; i < n; i++) {
            /* wrap-safe comparison: (int32_t)(now - due) >= 0 */
            if ((int32_t)(millis() - t[i].next_due) >= 0) {
                t[i].runs++;
                t[i].next_due += t[i].period;              /* no drift */
                strcat(log, t[i].name); strcat(log, " ");
            }
        }
    }

    /* Debounce: the stable level changes only after `need` identical consecutive samples. */
    typedef struct { bool stable; bool last; int count; } Deb;
    static bool debounce(Deb *d, bool sample, int need, bool *changed) {
        *changed = false;
        if (sample == d->last) { if (d->count < need) d->count++; } else { d->last = sample; d->count = 1; }
        if (d->count >= need && d->stable != d->last) { d->stable = d->last; *changed = true; }
        return d->stable;
    }

    int main(void) {
        Task tasks[] = { {"blink", 500, 500, 0}, {"sensor", 2000, 2000, 0}, {"report", 5000, 5000, 0} };
        char log[256] = "";
        fake_ms = 0xFFFFFFFFu - 700;                       /* start near the 32-bit millis() rollover */
        for (int i = 0; i < 3; i++) tasks[i].next_due = fake_ms + tasks[i].period;
        for (int step = 0; step < 12; step++) { fake_ms += 1000; poll(tasks, 3, log); }
        printf("%s\n", log);
        printf("runs: blink=%d sensor=%d report=%d\n", tasks[0].runs, tasks[1].runs, tasks[2].runs);

        int samples[] = {0,1,0,0,1,1,1,1,0,1,0,0,0,0};
        Deb d = {false, false, 0};
        for (unsigned i = 0; i < sizeof samples / sizeof *samples; i++) {
            bool ch; debounce(&d, samples[i], 3, &ch);
            if (ch) printf("sample %u: %s\n", i, d.stable ? "PRESSED" : "RELEASED");
        }
        return 0;
    }
    ```

    Comparing with (int32_t)(now - due) &gt;= 0 keeps scheduling correct when millis() wraps after ~49 days, and advancing next_due by the period avoids drift. Counting stable samples filters contact bounce without delay(). Verified by compiling and running the solution with gcc (host-side logic).

### Ring buffer, clamped map and ADC divider maths

Implement ring_push (overwrite oldest), ring_avg and map_clamped, then use the provided adc_to_volts to convert readings across a 220k/100k divider. Run the readings through the buffer to see how averaging smooths a noisy spike.

Starter code:

```
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

/* Ring buffer of readings with oldest-overwrite, plus a moving average and a smoothing ADC conversion. */
#define N 4
typedef struct { int v[N]; int head, count; } Ring;

static void ring_push(Ring *r, int x) {
    /* store x at head, advance head modulo N, and grow count up to N (the oldest value is overwritten) */
}
static double ring_avg(const Ring *r) {
    /* average of the stored values (0 when empty) */
    return 0;
}
static long map_clamped(long x, long in_min, long in_max, long out_min, long out_max) {
    /* Arduino map() but clamp x into [in_min, in_max] first */
    return 0;
}
/* 10-bit ADC reading across a divider: A0 sees Vin * 100k / (220k + 100k); Vin = 3.3 V max reading ~ 1023 at 1.0 V */
static double adc_to_volts(int raw) { return raw / 1023.0 * 1.0 * (220.0 + 100.0) / 100.0; }

int main(void) {
    Ring r = {{0}, 0, 0};
    int raw[] = {500, 520, 510, 900, 505, 515};
    for (unsigned i = 0; i < sizeof raw / sizeof *raw; i++) {
        ring_push(&r, raw[i]);
        printf("after %u samples: count=%d avg=%.1f\n", i + 1, r.count, ring_avg(&r));
    }
    printf("map 512 -> %ld, below range -> %ld, above range -> %ld\n",
           map_clamped(512, 0, 1023, 0, 100), map_clamped(-5, 0, 1023, 0, 100), map_clamped(5000, 0, 1023, 0, 100));
    printf("raw 1023 = %.2f V, raw 512 = %.2f V\n", adc_to_volts(1023), adc_to_volts(512));
    return 0;
}
```

Expected output:

```text
after 1 samples: count=1 avg=500.0
after 2 samples: count=2 avg=510.0
after 3 samples: count=3 avg=510.0
after 4 samples: count=4 avg=607.5
after 5 samples: count=4 avg=608.8
after 6 samples: count=4 avg=607.5
map 512 -> 50, below range -> 0, above range -> 100
raw 1023 = 3.20 V, raw 512 = 1.60 V
```

??? success "Show a solution"

    ```
    #include <stdio.h>
    #include <stdint.h>
    #include <stdbool.h>
    #include <string.h>

    /* Ring buffer of readings with oldest-overwrite, plus a moving average and a smoothing ADC conversion. */
    #define N 4
    typedef struct { int v[N]; int head, count; } Ring;

    static void ring_push(Ring *r, int x) {
        r->v[r->head] = x;
        r->head = (r->head + 1) % N;
        if (r->count < N) r->count++;
    }
    static double ring_avg(const Ring *r) {
        if (r->count == 0) return 0;
        int s = 0;
        for (int i = 0; i < r->count; i++) s += r->v[i];
        return (double)s / r->count;
    }
    static long map_clamped(long x, long in_min, long in_max, long out_min, long out_max) {
        if (x < in_min) x = in_min;
        if (x > in_max) x = in_max;
        return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min;
    }
    /* 10-bit ADC reading across a divider: A0 sees Vin * 100k / (220k + 100k); Vin = 3.3 V max reading ~ 1023 at 1.0 V */
    static double adc_to_volts(int raw) { return raw / 1023.0 * 1.0 * (220.0 + 100.0) / 100.0; }

    int main(void) {
        Ring r = {{0}, 0, 0};
        int raw[] = {500, 520, 510, 900, 505, 515};
        for (unsigned i = 0; i < sizeof raw / sizeof *raw; i++) {
            ring_push(&r, raw[i]);
            printf("after %u samples: count=%d avg=%.1f\n", i + 1, r.count, ring_avg(&r));
        }
        printf("map 512 -> %ld, below range -> %ld, above range -> %ld\n",
               map_clamped(512, 0, 1023, 0, 100), map_clamped(-5, 0, 1023, 0, 100), map_clamped(5000, 0, 1023, 0, 100));
        printf("raw 1023 = %.2f V, raw 512 = %.2f V\n", adc_to_volts(1023), adc_to_volts(512));
        return 0;
    }
    ```

    A fixed-size ring buffer bounds memory, averaging smooths spikes, and clamping before mapping stops out-of-range readings producing out-of-range outputs. Verified by compiling and running the solution with gcc (host-side logic).

---

More free practice: the [NodeMCU/IoT Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
