# 07 · Serial Monitor Debugging

!!! note "Not flashed to hardware"
    Reasoned through against the documented `Serial` class API shared by
    both Arduino cores (`begin`, `print`, `println`, `printf` availability
    differences, `available`/`read` for input). Not compiled or flashed to
    physical hardware in this environment.

## Why Serial matters more here than on classic Arduino

On a NodeMCU/ESP32 project, the Serial Monitor isn't just a nice-to-have
debugging aid — it's often your *only* window into what the board is
doing, especially once WiFi is involved (Module 08 onward) where a single
silent failure ("did it connect or not?") can otherwise leave you guessing
for a long time. Building a habit of logging state transitions to Serial
now pays off heavily for the rest of this course.

## Baud rate: it must match on both ends

`Serial.begin(baudRate)` sets the communication speed. The Serial Monitor
window in the Arduino IDE has its own baud-rate dropdown that **must
match** whatever you passed to `Serial.begin()`, or you'll see garbled
characters (or nothing at all).

`115200` is the conventional choice for ESP8266/ESP32 projects — much
faster than the classic Arduino default of `9600`, and comfortably
supported by both chips' UART hardware.

```cpp
// basic-serial-hello.ino
void setup() {
  Serial.begin(115200);
  delay(100); // brief pause; on some boards the USB-serial bridge needs
              // a moment to enumerate before the very first bytes are
              // reliably captured by a monitor that just opened
  Serial.println("Boot complete.");
}

void loop() {
  Serial.print("Uptime (ms): ");
  Serial.println(millis());
  delay(1000);
}
```

`millis()` returns the number of milliseconds since the board last
started/reset, as an `unsigned long` — printing it every second is a cheap
"is my board alive and running my code, or has it crashed/rebooted" check
that's useful throughout this entire course.

## `print` vs `println` vs `printf`

- `Serial.print(x)` — writes `x` with no trailing newline.
- `Serial.println(x)` — writes `x` followed by a newline; almost always
  what you want for readable, one-value-per-line log output.
- `Serial.printf(fmt, ...)` — C-style formatted printing, available on
  **both** the ESP8266 and ESP32 Arduino cores (unlike classic AVR-based
  Arduino, which doesn't support `Serial.printf` at all). Extremely useful
  for combining several values into one readable line:

```cpp
// serial-printf-example.ino
void setup() {
  Serial.begin(115200);
}

void loop() {
  int sensorRaw = 512;      // stand-in for a real reading
  float voltage = 1.65;     // stand-in for a computed value
  Serial.printf("raw=%d  voltage=%.2fV  uptime=%lums\n",
                sensorRaw, voltage, millis());
  delay(1000);
}
```

## Reading input from the Serial Monitor

The Serial Monitor's input box (top of the window) can send text back to
the board — useful for simple runtime configuration without recompiling,
like triggering a test action or toggling a mode:

```cpp
// serial-input-echo.ino
void setup() {
  Serial.begin(115200);
  Serial.println("Type a command and press Enter (try: status, reset)");
}

void loop() {
  if (Serial.available() > 0) {
    String command = Serial.readStringUntil('\n');
    command.trim(); // strips the trailing \r and any stray whitespace

    if (command == "status") {
      Serial.printf("OK - uptime %lums\n", millis());
    } else if (command == "reset") {
      Serial.println("Restarting...");
      delay(200);
      ESP.restart(); // available on both ESP8266 and ESP32 cores
    } else if (command.length() > 0) {
      Serial.printf("Unknown command: %s\n", command.c_str());
    }
  }
}
```

`Serial.readStringUntil('\n')` blocks only until it sees a newline or times
out (default timeout is 1000 ms per the core's documented default), and
`Serial.available()` guards the call so `loop()` doesn't stall waiting for
input that may never come — checking `available()` first before reading is
the standard non-blocking pattern for Serial input.

`ESP.restart()` performs a full software reset, documented identically
across both cores — handy for testing boot-time code (like the WiFi
connection logic in Module 08) repeatedly without physically pressing the
board's reset button.

## A structured logging helper

As sketches grow, prefixing every log line with a consistent tag makes the
Serial Monitor's scrollback far easier to scan, especially once multiple
subsystems (WiFi, sensor, web server) are all printing at once in later
modules:

```cpp
// tagged-logging-helper.ino
void logInfo(const char* tag, const String& message) {
  Serial.printf("[%8lu] [%s] %s\n", millis(), tag, message.c_str());
}

void setup() {
  Serial.begin(115200);
  logInfo("BOOT", "Setup starting");
  logInfo("BOOT", "Setup complete");
}

void loop() {
  logInfo("LOOP", "tick");
  delay(2000);
}
```

## Exercise

1. Run the basic Serial hello sketch, open **Tools → Serial Monitor**, set
   its baud rate to `115200`, and confirm you see the uptime counting up
   once per second.
2. Deliberately mismatch the baud rate (set the monitor to `9600` while the
   sketch uses `115200`) and observe the garbled output, so you recognize
   this failure mode instantly in the future.
3. Run the `printf` example and extend it to also print a fake "status"
   string (e.g. `"OK"` or `"WARN"`) alongside the numbers on the same line.
4. Run the Serial input echo sketch, type `status` and press Enter, confirm
   the uptime response, then type `reset` and confirm the board restarts
   (uptime resets to near zero) and reprints the boot message.
5. Adapt the tagged-logging helper into its own sketch and use it (instead
   of raw `Serial.println`) for every log line in a copy of the button
   sketch from Module 04 — log a `"BTN"`-tagged line only on each new press,
   not on every loop iteration.
