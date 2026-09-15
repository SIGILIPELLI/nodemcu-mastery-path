---
description: "Certification & Regulatory Basics — A sketch running on a bare NodeMCU dev board on your desk needs no certification. The moment you sell a device…"
---

# Certification & Regulatory Basics

!!! note "Reference module"
    This module covers regulatory concepts, not device firmware — there
    is no code to reason through against hardware APIs here. Content
    reflects generally documented regulatory categories; specific
    requirements vary by product and must be confirmed with a
    certification lab or regulatory counsel before shipping a real
    product.

## Why this matters even for a hobby-turned-product

A sketch running on a bare NodeMCU dev board on your desk needs no
certification. The moment you sell a device containing that same
ESP8266/ESP32 module to someone else, it becomes a regulated radio
product in most jurisdictions — ignoring this isn't a paperwork
shortcut, it's a legal and safety exposure.

## The module vs. the product

Most ESP8266/ESP32 modules (e.g. ESP-WROOM-02, ESP32-WROOM-32) are sold
pre-certified by the module manufacturer for radio emissions (FCC in
the US, CE-RED in the EU, and similar bodies elsewhere) — but that
certification is documented as valid only when the module is
integrated per the manufacturer's reference design (antenna, shielding,
PCB layout guidelines in the module's datasheet). Deviating from the
reference design (a different antenna, a metal enclosure that wasn't
part of the tested configuration) can void the module's existing
certification and require re-testing the finished product.

## Key certification categories to know

- **FCC (United States)** — radio frequency emissions. A module carrying
  FCC ID certification can often be integrated under a "modular
  approval," letting the end product skip full re-certification if
  integration rules are followed — check the module datasheet's stated
  conditions.
- **CE (European Union)** — covers the Radio Equipment Directive (RED)
  for radio devices, plus EMC and safety directives depending on the
  product (e.g. if it has a mains power supply).
- **IC (Canada)**, similar modular-approval structure to the FCC.
- **Country-specific approvals** — some markets (e.g. certain countries
  requiring type approval for WiFi devices) require additional
  certification beyond FCC/CE even when using a pre-certified module.

## What triggers a full re-certification

Documented triggers that can require re-testing even with a
pre-certified module:

1. Changing or adding an antenna not covered by the module's approval.
2. Enclosing the module in a way not represented in the original test
   configuration (materials that affect RF or thermal behavior).
3. Adding other radios (Bluetooth, a separate LoRa module) that weren't
   part of the original module's tested configuration.
4. Modifying RF-adjacent firmware behavior in ways that change
   transmit power or duty cycle beyond what the module's certification
   covered (relevant if using low-level radio calls rather than the
   documented WiFi stack defaults).

## Non-radio regulatory concerns

- **Battery safety** (UN 38.3 for shipping lithium batteries by air, if
  your product ships with one installed).
- **Electrical safety** (UL/CE safety marks) if the product includes a
  mains-connected power supply rather than being purely battery/USB
  powered.
- **RoHS/WEEE** (EU) — restricted substances and end-of-life recycling
  obligations for electronics sold in the EU.
- **Data privacy regulations** (GDPR in the EU, similar laws elsewhere)
  if the device collects any personal data, which is a legal category
  separate from radio/electrical certification entirely.

## A practical path for a small-batch product

1. Choose a pre-certified module and follow its reference design
   exactly for antenna and enclosure clearance.
2. Confirm with the module manufacturer's documentation which "modular
   approval" conditions apply in your target markets.
3. If selling beyond a hobbyist quantity or into markets with stricter
   type-approval rules, budget for a compliance consultant or testing
   lab review before mass production — this is not something firmware
   changes alone can satisfy.
4. Keep a compliance folder: module datasheets, the manufacturer's
   modular-approval documentation, and your own enclosure/antenna
   design decisions, in case a regulator or retailer asks for evidence.

## How It Actually Works

Radio certification (FCC Part 15, CE RED, etc.) exists because these chips are, physically, intentional radiators — the same RF matching network and antenna design covered under device design directly determines the actual radiated power and spectral emissions a test lab measures with a calibrated spectrum analyzer in an anechoic chamber, and pre-certified modules (versus a bare chip soldered onto your own custom RF layout) are attractive specifically because the module vendor has already had that exact antenna/matching/shielding combination tested and certified, and integrating it under a modular-approval path lets you inherit that certification without repeating the RF testing yourself — but only if you don't alter the module's RF section (antenna, matching network, shielding) at all, since any physical change there invalidates the basis the original certification tested.

Emissions testing also covers unintentional radiation from your own board design — fast digital switching edges (SPI/I2C clock lines, PWM outputs) act as small unintentional antennas radiating harmonics of their switching frequency, and poor PCB layout (long unshielded traces, inadequate ground-plane return paths near high-speed signals) can push a design over regulatory emission limits even with a perfectly certified Wi-Fi module on board — which is the concrete reason "we used a certified module" doesn't automatically mean "the finished product is certified": the module's radio emissions are covered, but your own board's separate digital-noise emissions are not.

*(These examples were written and reasoned through at the register/protocol level but were not flashed to a physical board for this pass — verify timing-sensitive details against your exact chip datasheet before relying on them in production.)*

## Exercise

1. Look up (from a module datasheet, real or representative) what
   antenna type and clearance the manufacturer specifies as part of its
   FCC/CE modular approval, and explain why deviating from it can void
   that approval.
2. Explain the practical difference between "the module is certified"
   and "the product is certified," using the enclosure-material example
   above.
3. For a product with a 3.7V LiPo battery, identify which regulatory
   category (from the list above) applies specifically to shipping it,
   separate from any radio certification.
4. Name one non-radio regulatory concern that would apply to an IoT
   product even if it had no wireless radio at all.
