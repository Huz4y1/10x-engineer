---
tags: [project, cpp, telemetry, realtime, idea]
status: idea
---

# Live F1 racing telemetry in C++

Index: [[App projects]] — Systems

---

## The idea

Ingest the F1 game's live UDP telemetry stream in C++, decode the binary packets, and drive a live dashboard — lap times, tyre temperatures, fuel, delta to best.

*(Fill in the detail — this note exists so the link resolves.)*

## Why C++ is genuinely the right choice here

This is one of the rare cases where [[When to leave Python]] says *yes*:

- **60Hz UDP** — packets arrive every ~16ms
- **Binary packet decoding** — struct parsing, byte order, no allocation in the hot path
- **Latency floor matters** — a laggy telemetry display is useless
- Per-packet overhead genuinely dominates

## Concepts used

| Concept | Note |
|---|---|
| UDP sockets | [[Networking reference]] |
| Binary parsing, structs, pointers | [[C++]] · [[Pointers]] · [[Memory]] |
| Byte order / endianness | [[04 — COMPUTER SCIENCE]] |
| Ring buffers, no allocation in the hot loop | [[Algorithms and data structures reference]] |
| Build system | [[C++ for this stack]] — CMake, `-O3` |
| Serving to a UI | [[Networking reference]] — WebSockets |

## Build steps

- [ ] **1 — Receive raw UDP packets** and print their size. Confirm you're getting data.
- [ ] **2 — Decode the packet header** — the game publishes its packet spec.
- [ ] **3 — Decode one packet type** (car telemetry) into a struct.
- [ ] **4 — Ring buffer** so decoding never blocks receiving.
- [ ] **5 — Push to a UI** over WebSockets.
- [ ] **6 — Store a session** for after-the-fact analysis → and now it's a [[Deployment patterns|batch]] problem.

> **Watch for endianness and struct padding.** `#pragma pack` or explicit field-by-field reads — never `memcpy` a network buffer straight into a struct and hope. That's the classic binary-protocol bug.

## Related

[[App projects]] · [[C++ for this stack]] · [[Networking reference]] · [[The Racing Cage]] · [[33 — SYSTEMS PERFORMANCE]]
