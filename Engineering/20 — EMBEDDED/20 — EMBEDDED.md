---
tags: [moc, embedded]
---

# 20 — EMBEDDED

> Software that runs directly on hardware, with no operating system to hide behind.

**Why it matters:** This is where code meets physics. A variable maps to a voltage on a pin; a bug can burn out a motor.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

> Existing notes: [[Embedded]] · [[Hardware]] · [[ESP32 board overview]] · [[Embedded C vs normal C]]

## Chips

Microcontrollers vs microprocessors · [[ESP32 board overview]] · STM32 · ARM

## Peripherals

[[GPIO pins and what they do]] · ADC ([[Analog input and ADC (ESP32)]]) · DAC · [[PWM output (ESP32)]] · [[Interrupts (ESP32)]] · [[Timers (ESP32)]]

## Communication

[[Serial and UART (ESP32)]] · [[SPI (ESP32)]] · [[I2C (ESP32)]] · CAN · USB · Ethernet · [[WiFi (ESP32)]] · [[Bluetooth (ESP32)]] · [[MQTT]]

## Hardware

[[Sensors overview]] · actuators · [[DC Motors]] · [[Brushless Motors and ESCs]] · [[Servo Motors]] · [[Stepper Motors]] · [[Transistors and MOSFETs]] · [[Voltage Regulators]]

## Software

Memory · flash · RAM · [[Bit manipulation and registers]] · datasheets · firmware · bootloaders · RTOS · FreeRTOS · embedded Linux · [[Volatile and interrupt safety]] · [[Non-blocking timing]] · [[State machines in embedded C]] · [[Fixed point and avoiding floats]]

## How C touches hardware

A GPIO pin is a **bit in a memory-mapped register**. Writing to an address changes a voltage.

```c
#define GPIO_OUT  (*(volatile uint32_t*)0x3FF44004)
GPIO_OUT |= (1 << 2);    // set pin 2 high
```

`volatile` tells the compiler this memory can change outside the program — without it, the optimiser deletes your reads. See [[Volatile and interrupt safety]].

## Related
[[03 — PROGRAMMING]] ([[C]], [[C++]]) · [[21 — ROBOTICS]] · [[22 — CONTROL SYSTEMS]] · [[23 — AEROSPACE]] · [[Kafka]]
