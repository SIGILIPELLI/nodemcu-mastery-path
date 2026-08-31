# Local Data Logging (SPIFFS/LittleFS)

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP8266/ESP32 `LittleFS`
    library API (and the legacy `SPIFFS` API it superseded). Not
    compiled or flashed to physical hardware in this environment.

## Why log locally at all

A device that only publishes over WiFi loses every reading during an
outage. Logging to the chip's onboard flash as a append-only file gives
you a buffer that survives reboots and network gaps, which you can
drain to the cloud once connectivity returns.

## LittleFS is the documented default

Both the ESP8266 and ESP32 Arduino cores document **LittleFS** as the
recommended flash filesystem (SPIFFS is documented as deprecated on
ESP8266 and unsupported for new projects on ESP32) — it has better wear
leveling and doesn't require full-filesystem rewrites the way SPIFFS
did.

```cpp
// littlefs-init.ino
#include <LittleFS.h>

void setup() {
  Serial.begin(115200);

  // begin(true) on ESP32 formats on first mount if none exists;
  // ESP8266's begin() auto-formats an unformatted partition too —
  // both are documented behaviors.
  if (!LittleFS.begin(true)) {
    Serial.println("LittleFS mount failed");
    return;
  }
  Serial.println("LittleFS mounted");
}

void loop() {}
```

!!! warning "Partition scheme"
    LittleFS needs a flash partition reserved for it. On ESP32 this is
    the documented "Default" or "Minimal SPIFFS" partition scheme
    selected in Tools → Partition Scheme; on ESP8266 it's the documented
    "FS" size selected in Tools → Flash Size. Without a reserved
    partition, `begin()` returns `false`.

## Appending readings as CSV lines

```cpp
// append-log.ino
#include <LittleFS.h>

const char* LOG_PATH = "/readings.csv";

void appendReading(float tempC, float humidity) {
  // "a" opens for append, documented to create the file if missing
  File f = LittleFS.open(LOG_PATH, "a");
  if (!f) {
    Serial.println("Failed to open log for append");
    return;
  }
  f.printf("%lu,%.2f,%.2f\n", millis(), tempC, humidity);
  f.close(); // documented as required to flush buffered writes
}

void setup() {
  Serial.begin(115200);
  LittleFS.begin(true);
  appendReading(21.5, 48.0);
}

void loop() {}
```

## Draining the log once online

```cpp
// drain-log.ino
#include <LittleFS.h>

bool drainLogOverSerial() {
  File f = LittleFS.open("/readings.csv", "r");
  if (!f) return false;

  while (f.available()) {
    String line = f.readStringUntil('\n');
    Serial.println(line); // stand-in for publishMqtt(line), etc.
  }
  f.close();

  // Once every line has been forwarded successfully, truncate the
  // log by removing and letting the next appendReading() recreate it.
  LittleFS.remove("/readings.csv");
  return true;
}
```

Truncating only after every line is confirmed sent avoids losing data
if the network drops mid-drain — a naive `remove()` called before
confirmation would discard unsent readings.

## Bounding log size

Flash is finite, and an unbounded CSV eventually fills the partition.
Check size before appending and rotate when it crosses a threshold:

```cpp
const size_t MAX_LOG_BYTES = 64 * 1024;

void appendReadingBounded(float tempC, float humidity) {
  File check = LittleFS.open("/readings.csv", "r");
  size_t currentSize = check ? check.size() : 0;
  if (check) check.close();

  if (currentSize > MAX_LOG_BYTES) {
    // Simplest rotation: rename the old file as a backup, start fresh.
    LittleFS.remove("/readings.old.csv");
    LittleFS.rename("/readings.csv", "/readings.old.csv");
  }

  File f = LittleFS.open("/readings.csv", "a");
  if (f) {
    f.printf("%lu,%.2f,%.2f\n", millis(), tempC, humidity);
    f.close();
  }
}
```

`File::size()` and `LittleFS.rename()` are both documented core APIs
on ESP8266 and ESP32.

## Inspecting what's stored

```cpp
void listFiles() {
  File root = LittleFS.open("/");
  File file = root.openNextFile();
  while (file) {
    Serial.printf("%s  %u bytes\n", file.name(), file.size());
    file = root.openNextFile();
  }
}
```

`openNextFile()` is the documented directory-iteration API on both
cores (LittleFS on ESP8266/ESP32 presents a flat namespace by default,
so this walks every file at the mount root).

## Exercise

1. Write a sketch that appends a timestamped CSV line every 10 seconds
   using the millis()-timer pattern from the scheduling module.
2. Add the size-based rotation logic and explain why `rename()` before
   `open(..., "a")` is safer than truncating in place.
3. Implement `drainLogOverSerial()` so it only calls `LittleFS.remove()`
   after the *entire* file has been read successfully — handle the case
   where `Serial` (stand-in transport) becomes unavailable partway
   through.
4. Explain, from the documented behavior of `LittleFS.begin(true)`,
   what happens the very first time a fresh board boots with this
   sketch, versus every boot after that.
