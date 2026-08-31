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
