---
title: "Final Quiz"
description: "Test what you have learned across all four levels of NodeMCU/IoT Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What voltage do NodeMCU GPIO pins tolerate?",
    "options": [
      "12 V",
      "5 V",
      "Anything",
      "3.3 V only: use a level shifter or divider for 5 V signals"
    ],
    "correctIndex": 3,
    "explanation": "Protect the pins."
  },
  {
    "question": "What does Arduino's map() give for a 10-bit reading of 512 mapped to 0..100?\n```c\n#include <stdio.h>\n#include <stdint.h>\n#include <stdbool.h>\n#include <string.h>\nlong map(long x, long in_min, long in_max, long out_min, long out_max){ return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min; }\nint main(void){ printf(\"%ld %ld\", map(512, 0, 1023, 0, 100), map(1023, 0, 1023, 0, 100)); }\n```\n",
    "options": [
      "51 100",
      "50 100",
      "512 1023",
      "50 99"
    ],
    "correctIndex": 1,
    "explanation": "Integer arithmetic. Verified by compiling and running."
  },
  {
    "question": "What duty cycle is analogWrite(256) on a 0..1023 scale?\n```python\nprint(round(256*100/1023,1))\n```\n",
    "options": [
      "12.5",
      "25.0",
      "100.0",
      "50.0"
    ],
    "correctIndex": 1,
    "explanation": "Value divided by range. Verified by running it."
  },
  {
    "question": "The board boots only when a sensor is unplugged. What's a likely cause?",
    "options": [
      "Bad Wi-Fi",
      "Low RAM",
      "USB",
      "The sensor loads a strapping pin (GPIO0/2/15) to the wrong level at boot"
    ],
    "correctIndex": 3,
    "explanation": "Check boot-mode pins."
  },
  {
    "question": "Why does `delay(10000)` in loop() hurt a Wi-Fi device?",
    "options": [
      "It saves power",
      "Style",
      "Law",
      "It blocks everything including background Wi-Fi/network handling; use millis()"
    ],
    "correctIndex": 3,
    "explanation": "Cooperative design."
  },
  {
    "question": "Garbage characters appear in the serial monitor. What do you check first?",
    "options": [
      "The sensor",
      "The cable color",
      "That the monitor baud rate matches Serial.begin()",
      "The router"
    ],
    "correctIndex": 2,
    "explanation": "Boot ROM messages use 74880."
  },
  {
    "question": "A DHT11 read returns NaN. What should the code do?",
    "options": [
      "Detect it (isnan), skip or retry and avoid sending bad data",
      "Reset",
      "Send NaN",
      "Use 0"
    ],
    "correctIndex": 0,
    "explanation": "Handle failures."
  },
  {
    "question": "The device must survive broker and Wi-Fi outages for hours. What design?",
    "options": [
      "Reconnect with backoff and buffer readings in a bounded ring buffer",
      "Reboot loop",
      "Block",
      "Drop all"
    ],
    "correctIndex": 0,
    "explanation": "Resilience."
  },
  {
    "question": "What does MQTT's Last Will do?",
    "options": [
      "Compresses",
      "Retries",
      "Tells the broker to publish a message if the client disconnects unexpectedly",
      "Encrypts"
    ],
    "correctIndex": 2,
    "explanation": "Presence/offline detection."
  },
  {
    "question": "Wake 3 s at 80 mA every 5 min, sleeping 297 s at 0.02 mA: average current in mA?\n```python\nprint(round((3*80+297*0.02)/300,3))\n```\n",
    "options": [
      "0.82",
      "8",
      "80",
      "0.02"
    ],
    "correctIndex": 0,
    "explanation": "Duty-cycled average. Verified by running it."
  },
  {
    "question": "For timer wake from deep sleep on an ESP8266, what must be wired?",
    "options": [
      "GPIO0 to GND",
      "TX to RX",
      "GPIO16 connected to RST",
      "A0 to 3V3"
    ],
    "correctIndex": 2,
    "explanation": "So the RTC can reset the chip."
  },
  {
    "question": "An OTA update is interrupted by a power cut. What prevents a brick?",
    "options": [
      "No checks",
      "Updating into a spare slot, verifying it and keeping a rollback path",
      "Retry blindly",
      "Overwrite in place"
    ],
    "correctIndex": 1,
    "explanation": "Safe updates."
  },
  {
    "question": "An ISR increments a counter read by loop(). What must you do?",
    "options": [
      "Use delay",
      "Declare it volatile, keep the ISR short and avoid Serial/delay inside it",
      "Nothing",
      "Use Serial"
    ],
    "correctIndex": 1,
    "explanation": "IRAM_ATTR on ESP."
  },
  {
    "question": "Two sensors on I2C don't respond. What do you do first?",
    "options": [
      "Reflash",
      "Change Wi-Fi",
      "Run an I2C scanner and check pull-ups, addresses and wiring",
      "Add delay"
    ],
    "correctIndex": 2,
    "explanation": "Debug the bus."
  },
  {
    "question": "Why validate and size ArduinoJson documents?",
    "options": [
      "RAM is small and oversized or malformed data can overflow or crash",
      "Speed",
      "Law",
      "Style"
    ],
    "correctIndex": 0,
    "explanation": "Bounded memory."
  },
  {
    "question": "setInsecure() works for HTTPS in the lab. Why not ship it?",
    "options": [
      "Not needed",
      "Larger",
      "Slow",
      "It skips certificate verification, enabling man-in-the-middle attacks"
    ],
    "correctIndex": 3,
    "explanation": "Pin a CA or fingerprint."
  },
  {
    "question": "The firmware hangs once a month in the field. What's a standard protection?",
    "options": [
      "Cron",
      "Print more",
      "Delay",
      "A hardware/software watchdog fed only when the system is healthy"
    ],
    "correctIndex": 3,
    "explanation": "Self-recovery."
  },
  {
    "question": "Every device shares one Wi-Fi/MQTT password in the firmware. What's the risk?",
    "options": [
      "None",
      "Speed",
      "Cost",
      "One leak compromises the whole fleet; provision unique per-device credentials"
    ],
    "correctIndex": 3,
    "explanation": "Revocation."
  },
  {
    "question": "You must roll out firmware to 10,000 devices. What strategy?",
    "options": [
      "All at once",
      "Manual",
      "Unsigned",
      "Staged (canary) rollout of signed images with health checks and automatic rollback"
    ],
    "correctIndex": 3,
    "explanation": "Limit blast radius."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A non-blocking scheduler and button debounce

Implement poll() so each task runs when its period has elapsed using a rollover-safe millis comparison (and no drift), and debounce() so the stable button level flips only after three identical consecutive samples. The program starts near the 32-bit millis() rollover.

**Starter code**

```cpp
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

/* Non-blocking scheduler driven by a fake millis() (like Arduino). Each task runs every `period` ms. */
typedef struct { const char *name; uint32_t period, next_due; int runs; } Task;
static uint32_t fake_ms = 0;
static uint32_t millis(void) { return fake_ms; }

typedef struct { bool stable; bool last; int count; } Deb;

static void poll(Task *t, int n, char *log) {
    /* run every task whose due time has been reached using the wrap-safe test (int32_t)(millis() - next_due) >= 0:
       increment runs, advance next_due by period (no drift) and append "<name> " to log */
}

static bool debounce(Deb *d, bool sample, int need, bool *changed) {
    /* *changed is true only on the sample where the stable level flips; the stable level changes only after
       `need` identical consecutive samples that differ from the current stable level */
    *changed = false;
    return d->stable;
}

int main(void) {
    Task tasks[] = { {"blink", 500, 500, 0}, {"sensor", 2000, 2000, 0}, {"report", 5000, 5000, 0} };
    char log[256] = "";
    fake_ms = 0xFFFFFFFFu - 700;                       /* start near the 32-bit millis() rollover */
    for (int i = 0; i < 3; i++) tasks[i].next_due = fake_ms + tasks[i].period;
    for (int step = 0; step < 12; step++) { fake_ms += 1000; poll(tasks, 3, log); }
    printf("%s\n", log);
    printf("runs: blink=%d sensor=%d report=%d\n", tasks[0].runs, tasks[1].runs, tasks[2].runs);

    int samples[] = {0,1,0,0,1,1,1,1,0,1,0,0,0,0};
    Deb d = {false, false, 0};
    for (unsigned i = 0; i < sizeof samples / sizeof *samples; i++) {
        bool ch; debounce(&d, samples[i], 3, &ch);
        if (ch) printf("sample %u: %s\n", i, d.stable ? "PRESSED" : "RELEASED");
    }
    return 0;
}
```

**Expected output**

```text
blink blink sensor blink blink sensor blink report blink sensor blink blink sensor blink blink sensor report blink blink sensor 
runs: blink=12 sensor=6 report=2
sample 6: PRESSED
sample 12: RELEASED
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdint.h&gt;
#include &lt;stdbool.h&gt;
#include &lt;string.h&gt;

/* Non-blocking scheduler driven by a fake millis() (like Arduino). Each task runs every `period` ms. */
typedef struct { const char *name; uint32_t period, next_due; int runs; } Task;
static uint32_t fake_ms = 0;
static uint32_t millis(void) { return fake_ms; }

static void poll(Task *t, int n, char *log) {
    for (int i = 0; i &lt; n; i++) {
        /* wrap-safe comparison: (int32_t)(now - due) &gt;= 0 */
        if ((int32_t)(millis() - t[i].next_due) &gt;= 0) {
            t[i].runs++;
            t[i].next_due += t[i].period;              /* no drift */
            strcat(log, t[i].name); strcat(log, &quot; &quot;);
        }
    }
}

/* Debounce: the stable level changes only after `need` identical consecutive samples. */
typedef struct { bool stable; bool last; int count; } Deb;
static bool debounce(Deb *d, bool sample, int need, bool *changed) {
    *changed = false;
    if (sample == d-&gt;last) { if (d-&gt;count &lt; need) d-&gt;count++; } else { d-&gt;last = sample; d-&gt;count = 1; }
    if (d-&gt;count &gt;= need &amp;&amp; d-&gt;stable != d-&gt;last) { d-&gt;stable = d-&gt;last; *changed = true; }
    return d-&gt;stable;
}

int main(void) {
    Task tasks[] = { {&quot;blink&quot;, 500, 500, 0}, {&quot;sensor&quot;, 2000, 2000, 0}, {&quot;report&quot;, 5000, 5000, 0} };
    char log[256] = &quot;&quot;;
    fake_ms = 0xFFFFFFFFu - 700;                       /* start near the 32-bit millis() rollover */
    for (int i = 0; i &lt; 3; i++) tasks[i].next_due = fake_ms + tasks[i].period;
    for (int step = 0; step &lt; 12; step++) { fake_ms += 1000; poll(tasks, 3, log); }
    printf(&quot;%s\n&quot;, log);
    printf(&quot;runs: blink=%d sensor=%d report=%d\n&quot;, tasks[0].runs, tasks[1].runs, tasks[2].runs);

    int samples[] = {0,1,0,0,1,1,1,1,0,1,0,0,0,0};
    Deb d = {false, false, 0};
    for (unsigned i = 0; i &lt; sizeof samples / sizeof *samples; i++) {
        bool ch; debounce(&amp;d, samples[i], 3, &amp;ch);
        if (ch) printf(&quot;sample %u: %s\n&quot;, i, d.stable ? &quot;PRESSED&quot; : &quot;RELEASED&quot;);
    }
    return 0;
}
</code></pre>

<p>Comparing with (int32_t)(now - due) &gt;= 0 keeps scheduling correct when millis() wraps after ~49 days, and advancing next_due by the period avoids drift. Counting stable samples filters contact bounce without delay(). Verified by compiling and running the solution with gcc (host-side logic).</p>
</details>

### Ring buffer, clamped map and ADC divider maths

Implement ring_push (overwrite oldest), ring_avg and map_clamped, then use the provided adc_to_volts to convert readings across a 220k/100k divider. Run the readings through the buffer to see how averaging smooths a noisy spike.

**Starter code**

```cpp
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

/* Ring buffer of readings with oldest-overwrite, plus a moving average and a smoothing ADC conversion. */
#define N 4
typedef struct { int v[N]; int head, count; } Ring;

static void ring_push(Ring *r, int x) {
    /* store x at head, advance head modulo N, and grow count up to N (the oldest value is overwritten) */
}
static double ring_avg(const Ring *r) {
    /* average of the stored values (0 when empty) */
    return 0;
}
static long map_clamped(long x, long in_min, long in_max, long out_min, long out_max) {
    /* Arduino map() but clamp x into [in_min, in_max] first */
    return 0;
}
/* 10-bit ADC reading across a divider: A0 sees Vin * 100k / (220k + 100k); Vin = 3.3 V max reading ~ 1023 at 1.0 V */
static double adc_to_volts(int raw) { return raw / 1023.0 * 1.0 * (220.0 + 100.0) / 100.0; }

int main(void) {
    Ring r = {{0}, 0, 0};
    int raw[] = {500, 520, 510, 900, 505, 515};
    for (unsigned i = 0; i < sizeof raw / sizeof *raw; i++) {
        ring_push(&r, raw[i]);
        printf("after %u samples: count=%d avg=%.1f\n", i + 1, r.count, ring_avg(&r));
    }
    printf("map 512 -> %ld, below range -> %ld, above range -> %ld\n",
           map_clamped(512, 0, 1023, 0, 100), map_clamped(-5, 0, 1023, 0, 100), map_clamped(5000, 0, 1023, 0, 100));
    printf("raw 1023 = %.2f V, raw 512 = %.2f V\n", adc_to_volts(1023), adc_to_volts(512));
    return 0;
}
```

**Expected output**

```text
after 1 samples: count=1 avg=500.0
after 2 samples: count=2 avg=510.0
after 3 samples: count=3 avg=510.0
after 4 samples: count=4 avg=607.5
after 5 samples: count=4 avg=608.8
after 6 samples: count=4 avg=607.5
map 512 -> 50, below range -> 0, above range -> 100
raw 1023 = 3.20 V, raw 512 = 1.60 V
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;stdio.h&gt;
#include &lt;stdint.h&gt;
#include &lt;stdbool.h&gt;
#include &lt;string.h&gt;

/* Ring buffer of readings with oldest-overwrite, plus a moving average and a smoothing ADC conversion. */
#define N 4
typedef struct { int v[N]; int head, count; } Ring;

static void ring_push(Ring *r, int x) {
    r-&gt;v[r-&gt;head] = x;
    r-&gt;head = (r-&gt;head + 1) % N;
    if (r-&gt;count &lt; N) r-&gt;count++;
}
static double ring_avg(const Ring *r) {
    if (r-&gt;count == 0) return 0;
    int s = 0;
    for (int i = 0; i &lt; r-&gt;count; i++) s += r-&gt;v[i];
    return (double)s / r-&gt;count;
}
static long map_clamped(long x, long in_min, long in_max, long out_min, long out_max) {
    if (x &lt; in_min) x = in_min;
    if (x &gt; in_max) x = in_max;
    return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min;
}
/* 10-bit ADC reading across a divider: A0 sees Vin * 100k / (220k + 100k); Vin = 3.3 V max reading ~ 1023 at 1.0 V */
static double adc_to_volts(int raw) { return raw / 1023.0 * 1.0 * (220.0 + 100.0) / 100.0; }

int main(void) {
    Ring r = {{0}, 0, 0};
    int raw[] = {500, 520, 510, 900, 505, 515};
    for (unsigned i = 0; i &lt; sizeof raw / sizeof *raw; i++) {
        ring_push(&amp;r, raw[i]);
        printf(&quot;after %u samples: count=%d avg=%.1f\n&quot;, i + 1, r.count, ring_avg(&amp;r));
    }
    printf(&quot;map 512 -&gt; %ld, below range -&gt; %ld, above range -&gt; %ld\n&quot;,
           map_clamped(512, 0, 1023, 0, 100), map_clamped(-5, 0, 1023, 0, 100), map_clamped(5000, 0, 1023, 0, 100));
    printf(&quot;raw 1023 = %.2f V, raw 512 = %.2f V\n&quot;, adc_to_volts(1023), adc_to_volts(512));
    return 0;
}
</code></pre>

<p>A fixed-size ring buffer bounds memory, averaging smooths spikes, and clamping before mapping stops out-of-range readings producing out-of-range outputs. Verified by compiling and running the solution with gcc (host-side logic).</p>
</details>
