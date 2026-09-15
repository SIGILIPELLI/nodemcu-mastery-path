---
description: "Fleet Management Concepts — One device you SSH-adjacent-debug by staring at Serial output. A fleet of thousands needs answers to questions no single…"
---

# Fleet Management Concepts

!!! note "Not flashed to hardware"
    Reasoned through against the documented MQTT protocol semantics
    (retained messages, Last Will and Testament) as exposed by the
    `PubSubClient` library on ESP8266/ESP32. Not compiled or flashed to
    physical hardware in this environment.

## From "a device" to "a fleet"

One device you SSH-adjacent-debug by staring at Serial output. A fleet
of thousands needs answers to questions no single device can answer by
itself: how many are online right now, which firmware versions are
deployed, which ones haven't reported in 24 hours. This module covers
the device-side hooks that make fleet-level visibility possible.

## Presence tracking with MQTT Last Will and Testament

MQTT brokers document a **Last Will and Testament (LWT)**: a message
the broker publishes automatically if a client disconnects
ungracefully (crash, power loss, network drop) without a normal
`DISCONNECT`. This is the standard mechanism for fleet-wide "is this
device actually online" tracking.

```cpp
// lwt-presence.ino
#include <PubSubClient.h>

WiFiClientSecure tlsClient;
PubSubClient mqtt(tlsClient);
String deviceId;

bool connectWithLWT() {
  String topic = "devices/" + deviceId + "/status";

  // connect(clientId, willTopic, willQos, willRetain, willMessage) is
  // the documented PubSubClient overload for setting an LWT.
  bool ok = mqtt.connect(deviceId.c_str(),
                          topic.c_str(), 1, true, "offline");
  if (ok) {
    // Publish "online" as a retained message immediately on connect —
    // retained means any dashboard subscribing later still sees the
    // current status without waiting for the next publish.
    mqtt.publish(topic.c_str(), "online", true);
  }
  return ok;
}
```

If the device crashes, the broker publishes `"offline"` to
`devices/<id>/status` on its behalf — no device-side code has to run for
the fleet dashboard to learn it went dark, which is exactly the
scenario (a hung device) where device-side code can't be trusted to run
anyway.

## Retained state topics for fleet inventory

Beyond presence, retaining the *last known* value of key fields on
well-known topics turns the broker into a lightweight fleet inventory
without a separate database round-trip on every dashboard load:

```cpp
// retained-fleet-state.ino
void publishFleetState() {
  String base = "devices/" + deviceId + "/";
  mqtt.publish((base + "firmware_version").c_str(), FW_VERSION, true);
  mqtt.publish((base + "ip").c_str(), WiFi.localIP().toString().c_str(), true);
  mqtt.publish((base + "rssi").c_str(), String(WiFi.RSSI()).c_str(), true);
  mqtt.publish((base + "last_seen").c_str(), String(millis()).c_str(), true);
}
```

A dashboard subscribing to `devices/+/firmware_version` (MQTT's
documented single-level wildcard `+`) receives every device's current
version immediately on subscribe, thanks to the retained flag — useful
for answering "how many devices are still on 1.3.0" without querying
each device live.

## Grouping devices for targeted commands

Fleet management usually needs to act on a subset — "reboot every
device in building B," not the whole fleet. Encode grouping into the
topic hierarchy so MQTT's wildcard subscriptions do the filtering:

```cpp
// grouped-command-topics.ino
String site = "building-b"; // set at provisioning time (Level 4.02)

void subscribeToCommands() {
  mqtt.subscribe(("devices/" + deviceId + "/cmd").c_str());  // per-device
  mqtt.subscribe(("sites/" + site + "/cmd").c_str());        // per-site
  mqtt.subscribe("fleet/cmd");                                 // all devices
}

void handleCommand(char* topic, byte* payload, unsigned int len) {
  String cmd((char*)payload, len);
  if (cmd == "reboot") {
    ESP.restart();
  } else if (cmd == "report_status") {
    publishFleetState();
  }
}
```

Publishing `"reboot"` to `sites/building-b/cmd` reaches only the
devices subscribed to that topic, without the server needing to track
individual device connections.

## Heartbeats as a fallback when LWT isn't enough

LWT depends on the broker detecting the disconnect (via its documented
keepalive timeout, typically the `keepalive` seconds passed to
`connect()`), which can lag reality by up to that interval. A
periodic heartbeat with a freshness check on the consuming side closes
that gap for anything time-sensitive:

```cpp
const unsigned long HEARTBEAT_INTERVAL_MS = 30000;
unsigned long lastHeartbeat = 0;

void loop() {
  mqtt.loop();
  unsigned long now = millis();
  if (now - lastHeartbeat >= HEARTBEAT_INTERVAL_MS) {
    lastHeartbeat = now;
    mqtt.publish(("devices/" + deviceId + "/heartbeat").c_str(),
                 String(now).c_str(), true);
  }
}
```

A server-side consumer that hasn't seen a device's heartbeat topic
update within, say, 3x the heartbeat interval can flag it as stale even
before the broker's own keepalive timeout fires.

## How It Actually Works

Fleet-wide device health signals are only meaningful because each device already exposes the same low-level hardware/firmware telemetry covered under diagnostics — reset reason register, free heap, RF signal strength (RSSI, a real analog measurement the radio's AGC/receiver front-end reports per received frame, not something firmware computes) — aggregated centrally rather than read one device at a time. A fleet dashboard showing a spike in brownout-reset devices in one geographic cohort is, mechanically, several thousand independent brownout comparators each independently tripping because of a shared root cause (a bad batch of regulators, or devices in a hotter climate zone drawing more current and sagging supply further) — the aggregation reveals a hardware population effect that would be invisible looking at any single device's logs.

Staged/canary OTA rollout across a fleet exploits the same rollback/pending-verify mechanism from firmware versioning, just orchestrated server-side: pushing a new image to 1% of devices first and watching their self-reported boot-confirmation and crash-reset rates before continuing the rollout is effectively using the fleet as a distributed hardware test bench, catching a firmware/hardware-revision incompatibility (a new sensor driver that behaves differently on a slightly different PCB revision, say) at 1% of blast radius instead of 100%, because the underlying rollback mechanism can only protect a device from its own bad firmware — it can't prevent a firmware bug from ever running in the first place, only limit how many devices experience it before the rollout halts.

*(These examples were written and reasoned through at the register/protocol level but were not flashed to a physical board for this pass — verify timing-sensitive details against your exact chip datasheet before relying on them in production.)*

## 🔀 Related lessons on other tracks

- [Embedded Linux — 07 · Fleet Management & Provisioning](https://sigilipelli.github.io/embedded-linux-mastery-path/level-4/07-fleet-management/)
- [Embedded — Fleet Management & Device Clouds](https://sigilipelli.github.io/embedded-mastery-path/level-4/09-fleet-management/)
- [Embedded Python — Fleet Management & Remote Monitoring](https://sigilipelli.github.io/embedded-python-mastery-path/level-4/07-fleet-management/)

## Exercise

1. Explain why the LWT message must be set at `connect()` time rather
   than published normally, based on the documented PubSubClient API
   shape.
2. Design the topic hierarchy for a fleet with three levels of grouping
   (fleet → region → site → device), and show the wildcard subscription
   a regional dashboard would use.
3. Implement a device-side handler for a `"set_report_interval:60"`
   command that reconfigures the heartbeat interval at runtime and
   persists it (via `Preferences`) so it survives reboot.
4. A device with a flaky WiFi connection could flap between "online"
   and "offline" many times per hour, generating LWT-then-reconnect
   noise on the fleet dashboard. Propose a device-side or
   dashboard-side mitigation.
