# Cost Engineering & BOM Optimization

!!! note "Reference module"
    This module covers bill-of-materials and cost tradeoffs, not
    firmware — code snippets here illustrate how a firmware decision
    (e.g. which chip family) trades off against BOM cost, reasoned
    against documented ESP8266/ESP32 capabilities rather than measured
    on hardware.

## The BOM is where margin is won or lost

At prototype quantities, a $2 difference in module cost is noise. At
10,000 units, it's $20,000 — often the difference between a profitable
product and one that isn't. Cost engineering means treating every
component choice as a tradeoff against what the firmware actually needs,
not defaulting to "the one I prototyped with."

## Chip family choice as a BOM decision

Level 4.01 framed ESP8266 vs. ESP32 as a capability question; at
volume it's also a straight cost question — the ESP8266 module is
documented as typically cheaper than the ESP32 module at comparable
volumes, so paying the ESP32 premium is only justified when the product
actually needs its extra GPIO, RAM, or BLE.

```cpp
// A firmware-level way to make this decision concrete: does the
// product's actual sketch fit ESP8266's documented RAM/GPIO ceiling?
//
// - 2 sensors over I2C, 1 status LED, MQTT over TLS, OTA:
//     fits comfortably within ESP8266's documented resources.
// - 5 sensors, local web dashboard, BLE provisioning, TLS + OTA + logging:
//     likely exceeds ESP8266's free-heap headroom once TLS buffers
//     and a web server are both resident — ESP32's RAM budget fits.
```

Making this comparison explicit (rather than assuming "we already
prototyped on ESP32") is often the single biggest per-unit cost lever
available before tooling is committed.

## Where cost hides beyond the main chip

- **Voltage regulator choice**: a linear regulator is cheaper than a
  switching regulator but wastes more power as heat — irrelevant for a
  USB-powered device, expensive (in wasted battery life, per Level
  4.05) for a battery product. The "right" regulator depends on the
  power budget you already modeled.
- **Antenna**: a PCB trace antenna is nearly free; an external antenna
  with a U.FL connector adds real per-unit cost and a certification
  variable (Level 4.08) if it changes the module's tested
  configuration.
- **Sensor grade**: a $0.30 generic temperature sensor vs. a $3
  factory-calibrated one is a real accuracy/cost tradeoff — worth
  revisiting once you know the product's actual required accuracy, not
  the accuracy that was convenient to prototype with.
- **Connectors and enclosure hardware**: often underestimated in a BOM
  built from a schematic alone; at volume, connector cost per unit
  frequently rivals the main chip's cost.

## A simple BOM cost model

```
per_unit_cost = chip_module + pcb_fab + passive_components
              + sensors + connectors + enclosure + assembly_labor

total_cost(qty) = per_unit_cost(qty) * qty + nre_costs
```

Where `per_unit_cost` itself typically drops with quantity (volume
pricing on the module and PCB fab), while `nre_costs` (non-recurring
engineering: tooling, certification testing from Level 4.08,
development time) are fixed regardless of unit count — meaning
cost-per-unit at 100 units can look very different from cost-per-unit
at 10,000, and a BOM decision that's "too expensive" at prototype scale
can be the right one at volume, or vice versa.

## Firmware choices that reduce BOM, not just power

- **Fewer external components via software**: using the chip's
  documented internal pull-up resistors (`pinMode(pin, INPUT_PULLUP)`)
  instead of a discrete pull-up resistor on every button/switch input
  removes a component per input, multiplied across the whole product
  line.
- **Sharing one ADC across multiple analog sensors** via an external
  multiplexer only pays off if it's cheaper than simply choosing an
  ESP32 (with its multiple ADC-capable pins) over an ESP8266 (with
  one) — another instance of the chip-choice tradeoff above.
- **Software debouncing** (Level 2.05) instead of a hardware RC filter
  per button removes passive components at zero firmware-complexity
  cost, since the debounce logic is a few lines already needed anyway.

## A BOM review checklist before committing to production

1. Does every component actually get used by the shipped firmware, or
   is something left over from prototyping (an unused sensor, a debug
   header) that could be removed?
2. Can any discrete component be replaced by a documented
   chip-internal feature (pull-ups, internal ADC reference, PWM instead
   of an external timer chip)?
3. Is the chip family (ESP8266 vs. ESP32) still the right choice given
   the *final* feature set, not the prototype's feature set?
4. Have connector, enclosure, and assembly labor costs been estimated
   from an actual quote, not extrapolated from the schematic alone?

## Exercise

1. For a product needing 3 digital button inputs, compare the BOM cost
   and firmware cost of external pull-up resistors vs.
   `INPUT_PULLUP`, and explain why the software option is very likely a
   pure win here.
2. Explain why the ESP8266's single ADC pin can turn into a hidden BOM
   cost (external multiplexer or ADC chip) for a product needing
   several analog sensors, and how that compares against simply
   choosing the ESP32.
3. Using the cost model above, explain why a design with high `nre_costs`
   but low `per_unit_cost` favors high-volume production, while the
   reverse favors a low-volume/niche product.
4. Identify one firmware decision from earlier in this level (OTA
   strategy, power budget, or security hardening) that has a *BOM*
   consequence, and describe it.
