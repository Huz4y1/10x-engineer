---
tags: [moc, c, libraries]
---

# C libraries

Language: [[C]] · Where it's used: [[20 — EMBEDDED]]

---

## Standard library

| Header | Gives you |
|---|---|
| `<stdio.h>` | `printf`, `scanf`, file I/O |
| `<stdlib.h>` | `malloc`, `free`, `atoi`, `rand` |
| `<string.h>` | `strcpy`, `strlen`, `memcpy`, `memset` |
| `<math.h>` | `sqrt`, `pow`, `sin` — link with `-lm` |
| `<stdint.h>` | **`uint8_t`, `int32_t`** — fixed-width types |
| `<stdbool.h>` | `bool`, `true`, `false` |
| `<time.h>` | Time |
| `<assert.h>` | `assert` |

> **On embedded, always use `<stdint.h>` types.** `int` is not a guaranteed size across platforms; `uint8_t` is. See [[Bit manipulation and registers]].

## Embedded

| Framework | For | Note |
|---|---|---|
| **ESP-IDF** | ESP32 native SDK | [[ESP32 board overview]] |
| **Arduino core** | Simpler ESP32/AVR API | [[Setting up the toolchain (ESP32)]] |
| **FreeRTOS** | Real-time tasks and scheduling | [[20 — EMBEDDED]] |
| HAL / CMSIS | STM32 and ARM | [[20 — EMBEDDED]] |

## Utility

| Library | For |
|---|---|
| **cJSON** | JSON on constrained devices |
| **libcurl** | HTTP |
| **SQLite** | Embedded database |
| Unity / CMock | Embedded testing |

> **Avoid `malloc` in embedded code.** Heap fragmentation on a device with 500 KB of RAM causes failures hours into a run. Prefer static allocation ([[Fixed point and avoiding floats]], [[Volatile and interrupt safety]]).

## Related

[[C]] · [[20 — EMBEDDED]] · [[Embedded C vs normal C]] · [[Memory (C)]]
