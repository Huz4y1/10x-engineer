The ESP32 is a microcontroller with WiFi and Bluetooth built into the same chip, which is why it took over the hobby world.

What is actually inside

|Thing|What you get|
|---|---|
|CPU|Two Xtensa LX6 cores at up to 240 MHz|
|RAM|520 KB SRAM, all of it, no more|
|Flash|Usually 4 MB on a DevKit, holds your program plus files|
|WiFi|802.11 b/g/n, client or access point|
|Bluetooth|Classic BT and BLE (not at the same time as easily)|
|GPIO|Up to 34 pins, but far fewer are usable, see [[GPIO pins and what they do]]|
|Logic level|3.3V. Not 5V. 5V on a GPIO kills the pin|
|Power|Board takes 5V over USB, an onboard regulator makes the 3.3V|

The peripherals and what each one is for

|Peripheral|Use it for|Note|
|---|---|---|
|GPIO|LEDs, buttons, relays|[[Digital input and output (ESP32)]]|
|ADC|Reading a pot, an LDR, a battery voltage|12-bit, [[Analog input and ADC (ESP32)]]|
|DAC|Actually outputting a voltage|Only GPIO 25 and 26, 8-bit|
|LEDC (PWM)|Dimming, motor speed, servos|16 channels, [[PWM output (ESP32)]]|
|UART|Serial monitor, GPS modules, other MCUs|3 of them, [[Serial and UART (ESP32)]]|
|I2C|Sensors, OLED screens, RTCs|2 wires, many devices, [[I2C (ESP32)]]|
|SPI|SD cards, displays, fast ADCs|4 wires, much faster, [[SPI (ESP32)]]|
|Touch|Capacitive touch pads, also a wake source|10 pins|
|Hall sensor|Detecting a magnet|Built in, weak, mostly a novelty|
|RTC + ULP|Staying awake cheaply while asleep|[[Deep sleep and power saving (ESP32)]]|
|Timers|Precise periodic work|4 hardware timers, [[Timers (ESP32)]]|

Two cores, and what runs where

```cpp
/*
1. under Arduino, your setup() and loop() run on core 1
2. the WiFi and Bluetooth stacks run on core 0
3. you can pin your own task to a core if you want, FreeRTOS is already running
*/

void heavyTask(void *param) {
    for (;;) {
        // this runs forever on its own core
        vTaskDelay(1000 / portTICK_PERIOD_MS);   // yield, never busy-wait here
    }
}

void setup() {
    Serial.begin(115200);
    Serial.println(xPortGetCoreID());   // output: 1

    xTaskCreatePinnedToCore(
        heavyTask,      // function
        "heavy",        // name, for debugging
        4096,           // stack in bytes
        NULL,           // parameter
        1,              // priority
        NULL,           // task handle
        0);             // core 0
}

void loop() {}
```

Compared to an Arduino Uno

|                |Arduino Uno|ESP32|
|---|---|---|
|Clock|16 MHz|240 MHz, dual core|
|RAM|2 KB|520 KB|
|Flash|32 KB|4 MB|
|Logic level|5V, tolerant and forgiving|3.3V, unforgiving|
|ADC|10-bit (0-1023)|12-bit (0-4095), and non-linear|
|Networking|none|WiFi + Bluetooth|
|PWM|analogWrite on 6 pins|16 LEDC channels, configurable frequency|
|OS|none at all|FreeRTOS is running underneath you|
|Price|about the same|about the same|

The board layout in rough terms

```
                    USB
                     │
   ┌─────────────────┴──────────────────┐
   │  ┌────────┐   ┌──────────────────┐ │
   │  │ CP2102 │   │                  │ │
   │  │  USB   │   │   ESP32-WROOM    │ │  ← metal can holds the
   │  │ serial │   │   CPU+RAM+flash  │ │    chip, flash and the
   │  └────────┘   │      + antenna   │ │    PCB antenna
   │               └──────────────────┘ │
   │  [EN]                       [BOOT] │
   │  ├─ 3V3  GND  ...pins...      GND ─┤
   └────────────────────────────────────┘

   The antenna end should hang off the edge of a breadboard,
   metal underneath it wrecks the WiFi range.
```

The one rule that saves hardware

```
  3.3V logic.

  A 5V sensor output wired straight to a GPIO can destroy that pin,
  and sometimes the chip. Use a level shifter or a divider.

  VIN pin  = 5V in from USB, feeds the regulator
  3V3 pin  = regulator output, ~600mA max, don't run motors off it
```
