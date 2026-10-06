---
tags: [project, embedded, iot, data-pipeline, applied]
status: not-started
---

# Project 04 - Smart Home Energy and Water Tracker

Track: [[Machine Learning Research Engineer]]

---

## What you're building

Real sensors in your home measuring electricity and water use, streaming into a pipeline, with a dashboard and a model that forecasts usage and flags anomalies.

**The first project in this track with real hardware and real consequences** — the data is about your actual bills.

## Why

Projects 01-03 were algorithms in isolation. This is a complete applied system: sensing → transport → storage → modelling → serving → monitoring. It's the architecture in [[Master architecture]] at household scale.

## Concepts used

| Layer | Concept | Note |
|---|---|---|
| Sensing | ESP32, ADC, interrupts | [[20 — EMBEDDED]] · [[ESP32 board overview]] |
| Sensors | CT clamp (current), flow meter (pulses) | [[Sensors overview]] |
| Transport | MQTT | [[MQTT]] |
| Storage | Delta / Postgres | [[SeaweedFS]] · [[PostgreSQL reference]] |
| Processing | Batch aggregation | [[PySpark core]] |
| Model | Time-series forecasting, anomaly detection | [[Choosing your approach]] |
| Serving | FastAPI | [[FastAPI fundamentals]] |
| Dashboard | Streamlit | [[Streamlit vs Django]] |
| Monitoring | Freshness, drift | [[Observability for data and ML pipelines]] |

## Hardware

| Part | Measures | Note |
|---|---|---|
| ESP32 | The brain | [[ESP32 board overview]] |
| **Non-invasive CT clamp** (SCT-013) | Current on the mains | ⚠️ Clamps *around* the cable — no mains contact |
| Water flow sensor (YF-S201) | Pulses per litre | [[Interrupts (ESP32)]] |
| DS18B20 | Temperature | Correlates with heating |

> **⚠️ Electrical safety.** Use a **non-invasive** CT clamp that closes around the outside of a cable. Do not open your consumer unit or touch mains wiring. If a project needs you inside the fuse box, it needs an electrician.

## Build steps

- [ ] **1 — One sensor, serial output.** Get a believable reading first.
- [ ] **2 — Calibrate.** Compare against your actual meter over an hour. **A sensor you haven't calibrated is a random number generator.**
- [ ] **3 — Count flow pulses with an interrupt**, not polling ([[Interrupts (ESP32)]]).
- [ ] **4 — Publish over MQTT**, non-blocking, with a Last Will ([[MQTT]]).
- [ ] **5 — Store it.** Postgres is plenty at this volume — don't reach for Kafka ([[Deployment patterns]]).
- [ ] **6 — Aggregate.** Hourly and daily kWh and litres.
- [ ] **7 — Baseline first.** "Tomorrow ≈ same weekday last week." **Beat this or you have no model** ([[Problem framing]]).
- [ ] **8 — Forecast.** Features: hour, day-of-week, temperature, recent usage.
- [ ] **9 — Anomaly detection.** A running toilet or immersion heater left on shows up as a step change.
- [ ] **10 — Dashboard + alerts.**

## Checkpoints

- [ ] Sensor readings match your real meter within ~5%
- [ ] Data survives a WiFi dropout and a reboot
- [ ] The model beats the naive baseline
- [ ] An anomaly you *cause deliberately* (leave a tap running) gets flagged
- [ ] It has run unattended for a week

## What will go wrong

| Symptom | Cause | Fix |
|---|---|---|
| Readings wildly wrong | Not calibrated | Calibrate against the real meter |
| ESP32 reboots randomly | Brownout | Separate supply, common ground, decoupling caps |
| Missing flow pulses | Polling instead of interrupts | Use an interrupt |
| MQTT disconnects | `delay()` blocking the loop | [[Non-blocking timing]] |
| Model can't beat the baseline | Usage is genuinely habitual | That's a real finding — report it |
| Gaps in the data | Device offline | Detect and record gaps explicitly |

## Related

[[20 — EMBEDDED]] · [[MQTT]] · [[Project 008 — Physical Intelligent Engine Monitor]] (same shape, different domain) · [[Master architecture]] · [[Problem framing]]
