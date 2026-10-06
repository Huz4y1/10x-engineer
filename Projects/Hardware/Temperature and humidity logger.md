A DHT22 reading room temperature and humidity, printed to serial and kept in a rolling buffer with min/max/average.

What you'll learn

- Talking to a sensor over a digital protocol instead of raw voltage
- Using a driver library rather than bit-banging
- Handling reads that fail — because they will
- Keeping a history in limited RAM

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|Breadboard + wires|1|£7|Yes|
|DHT22 (AM2302) sensor|1|£4|No|
|10kΩ resistor|1|£0.05|Yes|

DHT11 is the cheaper blue one and works with the same code (`DHT11` instead of `DHT22`), but it's ±2°C and integer-only. DHT22 is ±0.5°C with decimals — worth the extra £2. Some DHT22 modules come on a small PCB with the pull-up already fitted and only 3 pins; if so, skip the resistor.

Wiring

```
  3V3 ──────────┬─────────────────┐
                │                 │
              [10kΩ]              │
                │                 │
     ┌──────────┴─── ESP32 GPIO4  │  DATA (pulled up)
     │                            │
  ┌──┴──────────────┐             │
  │     DHT22       │             │
  │ VCC  DATA  NC  GND            │
  └──┬────────────┬──┘            │
     │            │               │
     └────────────┼───────────────┘
                  │
                 GND ───────────── ESP32 GND
```

|From|To|Note|
|---|---|---|
|DHT22 pin 1 (VCC)|ESP32 3V3|3.3V is fine, don't use 5V|
|DHT22 pin 2 (DATA)|ESP32 GPIO4|the single data line|
|DHT22 pin 2 (DATA)|10kΩ resistor leg 1|pull-up|
|10kΩ resistor leg 2|ESP32 3V3|holds the line HIGH when idle|
|DHT22 pin 3|nothing|not connected|
|DHT22 pin 4|ESP32 GND|common ground|

Pin 1 is the leftmost with the grille facing you. Getting VCC and GND backwards will make the sensor get noticeably hot within seconds — if you smell anything, unplug immediately.

How it works

Unlike the potentiometer, this isn't a voltage you measure. The DHT22 has its own little microcontroller inside. It measures a capacitive humidity element and a thermistor, and then sends you the result as 40 bits of digital data down a single wire, using pulse widths to encode 1s and 0s.

That one wire is bidirectional — the ESP32 pulls it low to say "give me a reading", releases it, then listens while the sensor drives it. When neither side is driving, the 10kΩ pull-up holds it HIGH so it never floats. Same idea as the button pull-up in [[Button with Pull-up and Pull-down]], just used as a shared bus.

Decoding those pulse widths by hand is miserable and timing-sensitive, so we use the Adafruit library. That's the normal path with real sensors — see [[Sensors overview]].

Two practical notes: the DHT22 will only give you a fresh reading every 2 seconds, and roughly 1 in 20 reads fails the checksum and comes back as NaN. Both are expected, and the code deals with both.

Install via Library Manager: **DHT sensor library** by Adafruit, plus **Adafruit Unified Sensor** which it depends on.

Code

```cpp
/*
1. the library handles the pulse-width protocol for us
2. reads are slow (2s minimum) and sometimes fail, so we check for NaN
3. a circular buffer keeps the last 60 samples without growing forever
4. min / max / average are computed over whatever is in the buffer
5. everything is non-blocking so other work could run alongside
*/

#include <DHT.h>

#define DHT_PIN  4
#define DHT_TYPE DHT22        // use DHT11 if that's what you bought

DHT dht(DHT_PIN, DHT_TYPE);

const int HISTORY = 60;       // 60 samples * 5s = 5 minutes of history
float tempLog[HISTORY];
int   count = 0;              // how many slots are filled
int   head  = 0;              // where the next sample goes

unsigned long lastRead = 0;
const unsigned long READ_EVERY = 5000;   // DHT22 needs >=2000, 5s is relaxed

void record(float t) {
  tempLog[head] = t;
  head = (head + 1) % HISTORY;           // wrap around, oldest gets overwritten
  if (count < HISTORY) count++;
}

void printStats() {
  if (count == 0) return;

  float lo = tempLog[0], hi = tempLog[0], sum = 0;

  for (int i = 0; i < count; i++) {
    float v = tempLog[i];
    if (v < lo) lo = v;
    if (v > hi) hi = v;
    sum += v;
  }

  Serial.printf("  over %d samples -> min %.1f  max %.1f  avg %.1f\n",
                count, lo, hi, sum / count);
}

void setup() {
  Serial.begin(115200);
  dht.begin();
  Serial.println("warming up...");
  delay(2000);                            // sensor needs a moment after power-up
}

void loop() {
  if (millis() - lastRead < READ_EVERY) return;
  lastRead = millis();

  float t = dht.readTemperature();        // celsius
  float h = dht.readHumidity();           // percent

  // a failed read comes back as NaN, which fails every comparison
  if (isnan(t) || isnan(h)) {
    Serial.println("read failed, skipping");
    return;
  }

  // the library can also work out how hot it *feels*
  float feels = dht.computeHeatIndex(t, h, false);

  record(t);

  Serial.printf("%.1f C   %.1f %%RH   feels %.1f C\n", t, h, feels);
  printStats();
}
```

Make it better

- Add a timestamp from an NTP server so the log is actually dated — needs [[WiFi (ESP32)]]
- Write samples to SPIFFS or an SD card so they survive a reboot
- Light a red LED above a threshold and a blue one below it
- Serve the readings over WiFi instead of serial — that's the next project, [[WiFi sensor dashboard]]
- Put the ESP32 to sleep between reads to run it off a battery for weeks — [[Deep sleep and power saving (ESP32)]]
