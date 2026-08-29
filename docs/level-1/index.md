# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: get comfortable with the NodeMCU family of boards — ESP8266 and ESP32 —
using the Arduino IDE and the Arduino C++ framework. You'll set up the
toolchain, blink an LED, read and write digital and analog pins, generate
PWM, debug over the Serial monitor, join a WiFi network, and read a real
temperature sensor — finishing with a WiFi-connected sensor node that serves
its reading over a tiny built-in web page.

**A word on hardware.** This level assumes you'll eventually run these
sketches on a real NodeMCU (ESP8266) or ESP32 dev board with the Arduino IDE
and a USB cable. Every sketch here is written to compile cleanly against the
documented ESP8266 and ESP32 Arduino-core APIs and is reasoned through
carefully against that documentation — but **none of it has been flashed to
physical hardware in this environment**, so treat the "expected output"
notes as what the datasheets and core source say should happen, and verify
on your own board before relying on it for anything critical.

## Modules

1. [What Is NodeMCU (ESP8266 vs ESP32)](01-what-is-nodemcu.md)
2. [Setting Up the Arduino IDE for ESP Boards](02-arduino-ide-setup.md)
3. [First Blink Sketch](03-first-blink-sketch.md)
4. [GPIO Basics: Digital Read & Write](04-gpio-basics.md)
5. [Analog Input (ADC)](05-analog-input-adc.md)
6. [PWM Basics](06-pwm-basics.md)
7. [Serial Monitor Debugging](07-serial-monitor-debugging.md)
8. [Connecting to WiFi](08-connecting-to-wifi.md)
9. [Reading a Sensor (DHT11 Temperature)](09-reading-a-sensor-dht11.md)
10. [Capstone — WiFi Sensor Node & Web Server](10-capstone-wifi-sensor-node.md)

By the end of this level you'll be able to set up either an ESP8266 or ESP32
board in the Arduino IDE from scratch, drive digital and PWM outputs, read
digital and analog inputs, print debug information over Serial, join a WiFi
network in station mode, read a temperature/humidity sensor, and combine all
of it into a small standalone IoT device with its own local web page.
