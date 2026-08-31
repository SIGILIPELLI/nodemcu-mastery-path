# Security Hardening for Production IoT

!!! note "Not flashed to hardware"
    Reasoned through against the documented ESP32 `esp_flash_encrypt`/
    `esp_secure_boot`/NVS-encryption APIs and the ESP8266/ESP32
    `WiFiClientSecure` TLS API. Not compiled or flashed to physical
    hardware in this environment.

## Threats specific to a device you don't physically control

A cloud server sits in a datacenter you control access to. A field IoT
device sits on someone else's shelf, reachable by anyone with a
screwdriver and a UART adapter. The threat model changes accordingly:
physical access to the flash chip, the debug UART, and the local
network are all in scope.

## 1. Stop shipping `setInsecure()` in production

Every TLS example earlier in this path used
`WiFiClientSecure::setInsecure()` to skip certificate validation — fine
for a bench sketch, a real vulnerability in production because it
accepts any TLS certificate, including an attacker's, enabling
man-in-the-middle interception of MQTT credentials and OTA payloads.

```cpp
// pinned-ca-cert.ino
#include <WiFiClientSecure.h>

// The broker/server's CA certificate, PEM-encoded, embedded at build
// time — documented as the setCACert() input format on both cores.
const char* ROOT_CA PROGMEM = R"EOF(
-----BEGIN CERTIFICATE-----
... actual CA cert content goes here ...
-----END CERTIFICATE-----
)EOF";

WiFiClientSecure secureClient;

void setupTls() {
  secureClient.setCACert(ROOT_CA); // documented on both cores' WiFiClientSecure
}
```

`setCACert()` validates the server's certificate chain against the
provided CA, which is what actually defends against interception —
`setInsecure()` provides none of that guarantee regardless of the TLS
version or cipher used underneath.

## 2. Flash encryption and secure boot (ESP32)

The ESP32 documents two independent protections against a physical
attacker reading the flash chip directly:

- **Flash encryption** (`esp_flash_encrypt` subsystem): the running
  firmware and stored data are encrypted at rest using a key burned
  into eFuse (one-time-programmable, not re-readable by software) —
  documented to prevent copying the flash chip's contents and reading
  secrets or cloning the firmware.
- **Secure Boot**: the bootloader documented to verify a cryptographic
  signature on the application image before executing it, preventing a
  physically-flashed malicious firmware image from running even if an
  attacker has full flash write access.

Both are enabled via build-time configuration (`menuconfig`/`idf.py`
options surfaced through `Tools` menu entries in recent Arduino-ESP32
core releases) rather than runtime API calls, and both are documented
as **one-way**: enabling them is not reversible, and losing the
associated keys bricks the device's ability to accept new signed
images. This is a manufacturing-time decision, not something toggled
per-sketch.

## 3. NVS encryption for stored secrets

Even without full flash encryption, the ESP32's `Preferences`/NVS
layer documents an NVS-encryption option (a separate encryption key
partition) so per-device secrets (Level 4.02's provisioned credentials)
aren't stored in plaintext even if flash encryption isn't fully
enabled. This is weaker than full flash encryption (the firmware
binary itself remains unencrypted) but cheaper to adopt on an existing
product.

## 4. Disable or gate the debug UART/JTAG in the field

Both chips document UART-based programming/monitoring as always
available unless explicitly disabled. For a product where physical
tampering is a real threat, documented options include:
ESP32's eFuse-based **UART download mode disable** (irreversible,
prevents re-flashing over UART entirely — pair only with a working OTA
path, since it removes your own recovery method too) and, less
drastically, requiring a physical jumper/button combination to enter
programming mode rather than leaving it always accessible.

## 5. Validate everything that arrives over the network

Firmware that trusts MQTT command payloads or OTA URLs without
validation is exploitable even with TLS in place (TLS proves *who*
you're talking to, not that *what* they sent is safe):

```cpp
// validate-commands.ino
void handleCommand(char* topic, byte* payload, unsigned int len) {
  if (len > 256) return; // reject oversized payloads outright

  String cmd((char*)payload, len);
  // Allow-list, not a free-form eval of the payload.
  if (cmd == "reboot" || cmd == "report_status" || cmd.startsWith("set_interval:")) {
    executeCommand(cmd);
  } else {
    logRejectedCommand(cmd);
  }
}

void executeCommand(const String& cmd) {}
void logRejectedCommand(const String& cmd) {}
```

## 6. A minimal hardening checklist

1. `setCACert()` (or certificate pinning) everywhere `setInsecure()`
   was used during development.
2. Per-device credentials (Level 4.02), never one shared secret baked
   into every unit.
3. Command/config payloads validated against an allow-list, not parsed
   and trusted blindly.
4. Flash/NVS encryption enabled if the product's threat model includes
   physical access to units in the field.
5. OTA images signature-verified before flashing (secure boot on ESP32,
   or an application-level signature check on ESP8266 where hardware
   secure boot isn't available).

## Exercise

1. Explain, in terms of what each actually verifies, why `setCACert()`
   defends against MITM interception but a valid-looking but
   attacker-controlled OTA URL still needs separate mitigation.
2. Why is enabling ESP32 flash encryption described as irreversible,
   and what operational risk does that create if the eFuse key is lost
   before a device ships?
3. Extend `handleCommand()`'s allow-list to support a signed command
   format: `command|hmac`, verifying the HMAC against a per-device
   secret (from `Preferences`) before executing.
4. The ESP8266 has no documented secure-boot equivalent to the ESP32's.
   Propose an application-level mitigation (checked before applying an
   OTA update) that approximates signature verification using tools
   already covered in this path.
