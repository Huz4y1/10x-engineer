---
tags: [project, esp32, embedded, mqtt, sensors, c]
status: not-started
---

# Project 008 — Physical Intelligent Engine Monitor

Index: [[28 — PROJECTS]] · Previous: [[Project 007 — Cloud Deployment]]

---

## What you're building

Real hardware. An **ESP32** with real sensors on a real motor, publishing over **MQTT** into the pipeline you already built, with a model predicting failure from data **you generated yourself**.

## Why this project exists

Projects 001-007 used clean, pre-collected NASA data. Real sensors are noisy, drift with temperature, disconnect, and occasionally lie.

**Afterwards you will understand:** why data engineers are paranoid about data quality, what an ADC actually measures, why timestamps are harder than they look, and why the physical world is the hardest input.

> **This is the most valuable project in the sequence.** The step from clean public data to your own messy sensor data teaches more than the previous four combined.

## Concepts used

| Concept | Note | New? |
|---|---|---|
| GPIO, ADC, PWM, interrupts, timers | [[20 — EMBEDDED]] | New |
| Embedded C/C++ | [[C]] · [[C++]] · [[Embedded C vs normal C]] | New |
| Non-blocking loops | [[Non-blocking timing]] | New |
| Sensors and wiring | [[Sensors overview]] · [[Breadboards and wiring]] | New |
| MQTT | [[MQTT]] | New |
| Kafka bridge | [[Kafka]] | Revision |
| The whole pipeline | Projects 003-006 | Revision |

## Hardware

| Part | Purpose | Note |
|---|---|---|
| ESP32 dev board | The brain | [[ESP32 board overview]] |
| DHT22 or DS18B20 | Temperature | [[Sensors overview]] |
| MPU6050 (I²C) | Vibration / accelerometer | [[I2C (ESP32)]] |
| ACS712 or INA219 | Current draw | Motor load |
| Small DC motor + driver | The "engine" | [[DC Motors]] · [[H-Bridge motor driver]] |
| Breadboard, jumpers, PSU | — | [[Breadboards and wiring]] |

> **Vibration is the interesting signal.** A bearing wearing out changes its vibration signature long before it fails. That's genuinely how industrial predictive maintenance works.

## Architecture

```mermaid
flowchart LR
    M["DC motor"] --> S["Sensors<br/>temp, vibration, current"]
    S -->|ADC / I2C| E["ESP32<br/>C++"]
    E -->|MQTT / WiFi| B["Mosquitto"]
    B -->|bridge| K["Kafka"]
    K --> SP["PySpark"]
    SP --> D["Delta bronze/silver/gold"]
    D --> T["PyTorch"]
    T --> API["FastAPI"]
    API --> DASH["Dashboard"]
```

## Build steps

- [ ] **1 — Blink an LED**
  Prove the toolchain works end to end before adding anything ([[Setting up the toolchain (ESP32)]]).

- [ ] **2 — Read one sensor, print to serial**
  ```cpp
  void loop() {
      float t = dht.readTemperature();
      if (isnan(t)) { Serial.println("read failed"); return; }   // <- sensors DO fail
      Serial.printf("temp=%.2f\n", t);
      delay(2000);
  }
  ```
  > **Every sensor read can fail.** Handle it from the first line — this is the trust boundary, and NaN propagating into your pipeline is a genuinely painful bug to trace back.

- [ ] **3 — All sensors, non-blocking**
  ```cpp
  unsigned long lastRead = 0;
  const unsigned long INTERVAL = 1000;

  void loop() {
      mqtt.loop();                                  // MUST run constantly
      if (millis() - lastRead >= INTERVAL) {        // non-blocking
          lastRead = millis();
          readAndPublish();
      }
  }
  ```
  > **`delay()` blocks everything** — including MQTT keep-alives, so the broker decides you've died. Never use `delay()` in a loop that does anything else ([[Non-blocking timing]]).

- [ ] **4 — Publish over MQTT**
  ```cpp
  char payload[128];
  snprintf(payload, sizeof(payload),
      "{\"ts\":%lu,\"temp\":%.2f,\"vib_rms\":%.3f,\"current\":%.3f}",
      millis(), temp, vibRms, current);
  mqtt.publish("engine/1/telemetry", payload, false);
  ```
  With a **Last Will** so the dashboard knows within seconds when the device dies:
  ```cpp
  mqtt.connect("engine-01", user, pass, "engine/1/status", 1, true, "offline");
  mqtt.publish("engine/1/status", "online", true);
  ```

- [ ] **5 — The timestamp problem**
  `millis()` is milliseconds since **boot**, not wall-clock time. It resets on every reboot and drifts.
  ```cpp
  configTime(0, 0, "pool.ntp.org");     // sync real time over NTP
  time_t now; time(&now);
  ```
  > **Record both**: the device's own clock *and* the server's receive time. When they disagree you learn something — a reboot, a network stall, a clock drift. This is exactly why bronze keeps ingestion metadata.

- [ ] **6 — Bridge MQTT to Kafka**
  ```
  # mosquitto.conf
  connection kafka-bridge
  address localhost:9092
  topic engine/# out 0
  ```
  Or a small Python bridge subscribing to `engine/#` and producing to Kafka. Either way, **MQTT transports, Kafka stores** ([[MQTT]]).

- [ ] **7 — Handle real-world mess in silver**
  Your NASA cleaning rules won't cover this. Add:
  ```python
  from pyspark.sql import functions as F

  .filter(F.col("temp").between(-40, 150))          # physically possible range
  .filter(F.col("vib_rms") >= 0)
  .dropDuplicates(["device_id", "device_ts"])       # MQTT QoS 1 duplicates
  .filter(F.col("temp").isNotNull())                # failed reads
  ```
  Plus **gap detection** — a device that stopped reporting for ten minutes is a fact you need to record, not silently ignore.

- [ ] **8 — Generate failure data**
  You need examples of failure to predict failure. Safely induce degradation:
  - Increase load (add friction) progressively
  - Run to overheating (with a hard thermal cutoff in firmware)
  - Loosen a mount to increase vibration
  - **Label every run** with what you did and when

  > **This is the labelling project [[Problem framing]] warns about.** Without labelled failures you have no supervised learning problem, only a data collection exercise.

- [ ] **9 — Train and deploy**
  Same as Projects 004-005, on your own data. **Expect it to be much harder.** Real signal-to-noise is far worse than NASA's simulator.

## Checkpoints

- [ ] Data flowing device to dashboard, end to end
- [ ] Unplugging the ESP32 makes the dashboard show "offline" within seconds (Last Will)
- [ ] Bronze contains both device time and server time
- [ ] You can explain every row silver drops, and roughly how many
- [ ] You have labelled failure runs
- [ ] A model trained on your own data, honestly compared to a baseline

## Make it fail deliberately

- [ ] Unplug the WiFi mid-run — does it reconnect and resume?
- [ ] Use `delay(5000)` in the loop and watch MQTT disconnect
- [ ] Unplug a sensor and confirm NaN doesn't reach silver
- [ ] Reboot the device and see `millis()` reset in your data

## What will go wrong

| Symptom | Cause | Fix |
|---|---|---|
| Sensor reads NaN intermittently | Loose wiring, or timing too fast | Check connections; respect the sensor's minimum interval |
| ESP32 reboots randomly | Brownout — motor draws too much | Separate motor supply; common ground; decoupling caps |
| MQTT keeps disconnecting | Blocking loop, or keep-alive too short | Non-blocking; call `mqtt.loop()` often |
| Readings drift over hours | Thermal drift — genuinely real | Calibrate; log ambient temperature too |
| Timestamps nonsensical | `millis()` resets on reboot | NTP + record both clocks |
| Duplicate readings | MQTT QoS 1 | Deduplicate on `(device, device_ts)` |
| Noisy vibration data | No filtering | Low-pass filter ([[Noise and Filtering]]) |
| Model won't learn | Not enough failures; poor labels | More runs; better labelling |

> **Brownout is the one that wastes an evening.** A motor starting draws a large current spike, the rail sags, and the ESP32 resets. Separate supplies with a common ground, and a decoupling capacitor across the motor ([[Decoupling capacitors and clean power]], [[Flyback diode protection]]).

## Stretch goals

- [ ] FFT on-device and publish frequency bands instead of raw samples — far less bandwidth, better features
- [ ] Deep sleep between readings to run on battery ([[Deep sleep and power saving (ESP32)]])
- [ ] Run the model **on the device** (TensorFlow Lite Micro) and compare with cloud inference
- [ ] OTA firmware updates

## What you learned

*Fill in afterwards. This project will surprise you the most.*

## Next project

[[Project 009 — Autonomous Robotics System]]
