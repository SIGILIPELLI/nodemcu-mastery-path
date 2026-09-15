---
description: "Security Hardening for Production IoT — A cloud server sits in a datacenter you control access to. A field IoT device sits on someone else's shelf…"
---

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

## How It Actually Works

Secure boot and flash encryption, mentioned earlier under provisioning, are the two hardware mechanisms that actually close the physical attack surface a bare dev board leaves wide open: flash encryption uses a device-unique AES key burned into one-time-programmable eFuses (bits that, once set, cannot be reset or read back out by any software instruction, only used internally by the flash-encryption hardware engine) so that every read/write to flash is transparently encrypted/decrypted in hardware — an attacker who desolders the flash chip and dumps its raw contents gets ciphertext, not your firmware or secrets in the clear. Secure boot layers on top of this: each boot stage's digest is signed with a private key at build time, and the immediately-prior boot stage (starting from an immutable ROM bootloader burned at the factory) verifies that signature against a public key also burned into eFuses before it will execute the next stage at all, which is what actually prevents someone from just re-flashing arbitrary unsigned firmware onto a stolen device through the same UART bootloader interface every dev board exposes.

Once both are enabled, they also change your own update workflow at a hardware level, not just an attacker's: JTAG debugging can be permanently disabled via another eFuse (a genuinely irreversible hardware fuse blow, not a software toggle), and any new firmware image must be signed with the matching private key or secure boot will refuse to execute it — meaning losing your signing key doesn't just block your next OTA, it can permanently strand already-deployed hardware that refuses everything else, which is why key management for these fuses is a harder and higher-stakes operational problem than almost anything else in the firmware lifecycle.

*(These examples were written and reasoned through at the register/protocol level but were not flashed to a physical board for this pass — verify timing-sensitive details against your exact chip datasheet before relying on them in production.)*

## 🔀 Related lessons on other tracks

- [Cybersecurity — 03 · Linux Security Hardening](https://sigilipelli.github.io/cybersecurity-mastery-path/level-2/03-linux-security-hardening/)
- [Embedded Linux — 08 · Security Hardening & CVE Management](https://sigilipelli.github.io/embedded-linux-mastery-path/level-4/08-security-hardening/)
- [Embedded Python — Security Hardening — TLS & Secure Storage](https://sigilipelli.github.io/embedded-python-mastery-path/level-4/06-security-hardening/)

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
