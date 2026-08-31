# OTA Update Strategy at Scale

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP8266
    `ESP8266httpUpdate` and ESP32 `HTTPUpdate`/`esp_ota_ops` OTA APIs.
    Not compiled or flashed to physical hardware in this environment.

## What changes between "update one device" and "update a fleet"

Level 3.08 covered validating and rolling back a single device's OTA
update. At fleet scale, the new problems are: not overwhelming your
update server, not bricking everything at once with a bad build, and
knowing what's actually deployed across thousands of units.

## Staggered check-in windows to avoid a thundering herd

If every device checks for updates on a fixed interval starting from
power-on, a fleet-wide power event (e.g. a regional outage recovering)
makes them all check in within the same second, hammering the update
server. Jittering the check based on the device's own identity spreads
the load deterministically without coordination:

```cpp
// staggered-checkin.ino
unsigned long checkInIntervalMs() {
  // Derive a stable per-device jitter (0-300s) from the MAC address,
  // so the same device always checks in at the same offset, but
  // different devices are spread across a 5-minute window.
  String mac = WiFi.macAddress();
  uint32_t hash = 0;
  for (size_t i = 0; i < mac.length(); i++) hash = hash * 31 + mac[i];
  uint32_t jitterMs = hash % 300000UL;

  const unsigned long BASE_INTERVAL_MS = 3600000UL; // 1 hour
  return BASE_INTERVAL_MS + jitterMs;
}

unsigned long lastCheck = 0;
unsigned long nextInterval = 0;

void loop() {
  if (nextInterval == 0) nextInterval = checkInIntervalMs();
  if (millis() - lastCheck >= nextInterval) {
    lastCheck = millis();
    nextInterval = checkInIntervalMs(); // re-derive in case config changed
    checkForUpdate();
  }
}

void checkForUpdate() { /* Level 3.08's attemptUpdate() */ }
```

## Staged rollout using the version-check header

`ESPhttpUpdate`/`HTTPUpdate` on both cores document sending the
device's current firmware version as a request header
(`x-ESP8266-version` / equivalent) on every check. A server-side
staged rollout uses that plus the device ID to decide who gets the new
build:

```cpp
// (server-side pseudocode, not device code)
// GET /update  headers: x-ESP32-version: 1.4.2, x-device-id: gw-aabbcc...
//
// if device_id_hash(device_id) % 100 < rollout_percentage(new_version):
//     serve new binary
// else:
//     return 304 Not Modified   # documented HTTP_UPDATE_NO_UPDATES result
```

Because the rollout decision is a pure function of `device_id` and the
current `rollout_percentage`, increasing the percentage from 5% to 25%
to 100% over days doesn't require the server to remember which devices
were already selected — the same devices stay in the "yes" set as the
percentage grows, and no state needs to be tracked per device.

## Halting a bad rollout fast

Because Level 3.08's self-test-then-rollback pattern runs per-device,
a bad build doesn't need every device to individually detect the
problem before you act — the fleet's diagnostics telemetry (Level
3.09/4.03's retained-status topics) makes the failure visible in
aggregate quickly:

```cpp
// fleet-side signal a device already sends (from 4.03's heartbeat/state)
// devices/<id>/firmware_version  (retained)
// devices/<id>/status            (retained: online/offline via LWT)
//
// A dashboard subscribing to devices/+/firmware_version and
// devices/+/status can compute, within one heartbeat interval:
//   "of the 40 devices now on 1.5.0, 12 have gone offline" — a strong
// signal to set rollout_percentage back to 0 on the server immediately.
```

The device-side contribution to fast rollback detection is simply:
publish version and status promptly after every boot (already covered
in 4.03), so the fleet-level aggregation has fresh data to work with.

## Bandwidth: don't ship the whole binary to everyone, every time

For fleets on metered or constrained connections, two documented
levers reduce OTA bandwidth:

- **Delta/differential updates**: not built into the core OTA APIs, but
  achievable by having the server compute a binary diff against the
  device's reported current version and serving a smaller patch that a
  bootloader-side or application-side patcher reconstructs into the
  full image before handing it to `Update`/`ESP8266httpUpdate`. This is
  a build-pipeline decision, not a firmware-API one.
- **Compression in transit**: serving a gzip-compressed binary over
  HTTPS (most HTTP servers documented to support `Content-Encoding:
  gzip` transparently) shrinks the download without any device-side
  code change, since decompression happens at the TCP/TLS layer via
  the server and a gzip-aware client — note `HTTPUpdate` itself expects
  a raw firmware stream, so this only helps if your CDN/server
  decompresses before handing bytes to the update client, or your
  update client explicitly wraps the stream in a gzip decoder.

## A minimal fleet OTA policy, expressed as constants

```cpp
// ota-policy.h
#define OTA_CHECK_BASE_INTERVAL_MS   3600000UL  // 1 hour
#define OTA_CHECK_JITTER_MAX_MS      300000UL   // 5 minutes
#define OTA_SELF_TEST_TIMEOUT_MS     30000UL    // must validate within 30s
#define OTA_MAX_ROLLBACK_ATTEMPTS    1          // don't rollback-loop forever
```

Codifying these as named constants (rather than magic numbers scattered
through the update logic) makes a fleet-wide policy change — e.g.
tightening the self-test window after an incident — a one-line diff
instead of a hunt through the codebase.

## Exercise

1. Implement the MAC-based jitter function and verify by hand that two
   different MAC addresses map to check-in offsets more than 60 seconds
   apart (pick two MACs and trace the hash).
2. Explain why deriving the staged-rollout decision from a hash of
   `device_id` (rather than randomly picking devices each time the
   percentage changes) keeps the rollout monotonic as the percentage
   increases.
3. Using the retained MQTT state topics from 4.03, sketch the exact
   subscription and aggregation logic a "canary watchdog" service would
   run to auto-pause a rollout if more than 10% of updated devices go
   offline within 10 minutes of updating.
4. Why does `HTTPUpdate` expecting a raw firmware stream complicate
   using plain HTTP gzip compression for OTA, and what are the two ways
   around it named above?
