Getting code onto an ESP32 is mostly a one-off setup job, after that it's write, upload, watch the serial monitor.

Adding ESP32 board support to the Arduino IDE

```
  File > Preferences > Additional Board Manager URLs

  https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json

  then  Tools > Board > Boards Manager > search "esp32" > Install
```

Picking the board

|Your board says|Pick this in Tools > Board|
|---|---|
|ESP32 DevKit v1 / DOIT|DOIT ESP32 DEVKIT V1|
|NodeMCU-32S|NodeMCU-32S|
|WROOM-32 generic|ESP32 Dev Module|
|ESP32-S3 / C3|the matching S3 or C3 entry, they are different chips|
|no idea|ESP32 Dev Module, it works for most WROOM boards|

The USB driver problem, the single most common "it doesn't work"

```
  The ESP32 chip has no USB on the classic boards. A separate chip
  on the board converts USB to serial. You need ITS driver, not an
  "ESP32 driver".

  Look at the small square chip next to the USB socket:

    CP2102   ->  Silicon Labs CP210x VCP driver
    CH340 /
    CH9102   ->  WCH CH34x driver

  No port showing up in Tools > Port  =  driver missing, or you are
  using a charge-only USB cable with no data lines in it.
```

Uploading

```cpp
/*
1. this is the smallest program that proves the whole chain works
2. setup() runs once, loop() runs forever after that
3. LED_BUILTIN is usually GPIO 2 on a DevKit
*/

void setup() {
    pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
    digitalWrite(LED_BUILTIN, HIGH);
    delay(500);
    digitalWrite(LED_BUILTIN, LOW);
    delay(500);
}
```

The hold BOOT gotcha

```
  To flash, the chip must be in bootloader mode. Most boards do this
  automatically using the DTR/RTS lines. Some don't.

  If the upload log sits there printing dots:

    Connecting........_____....._____

  hold the BOOT (sometimes IO0) button down, keep holding until the
  log says "Writing at 0x...", then let go.

  Stubborn boards: hold BOOT, tap EN (reset), release BOOT.

   ┌───────────────────────────────┐
   │  [EN]                  [BOOT] │
   │   reset            pulls IO0  │
   │                     low       │
   │        ┌──────────┐           │
   │  USB   │ WROOM-32 │           │
   │  ┌─┐   └──────────┘           │
   └──┴─┴───────────────────────────┘
```

The serial monitor

```cpp
void setup() {
    Serial.begin(115200);      // 115200 is the ESP32 default, use it
    Serial.println("alive");
}

void loop() {}

// Tools > Serial Monitor, and set the dropdown to 115200 as well.
// Mismatched baud = screen full of  ÿÿ<Ã¾  nonsense.
```

The ESP-IDF way, if you outgrow Arduino

```bash
# ESP-IDF is Espressif's own framework. More setup, far more control.
# Arduino for ESP32 is actually a layer sitting on top of it.

idf.py set-target esp32
idf.py build
idf.py -p COM5 flash monitor    # flash and open the monitor in one go

# Ctrl+] quits the IDF monitor
```

PlatformIO is the middle ground, a VS Code extension, one config file

```
  platformio.ini

  [env:esp32dev]
  platform = espressif32
  board = esp32dev
  framework = arduino
  monitor_speed = 115200
```

Quick fault list

|Symptom|Cause|
|---|---|
|No port in the menu|USB driver not installed, or charge-only cable|
|Connecting......  timeout|Hold BOOT while uploading|
|Uploads fine, then reboots forever|Something wired to a strapping pin, see [[GPIO pins and what they do]]|
|Garbled serial output|Baud rate mismatch|
|Brownout detector was triggered|Weak USB port or a motor pulling too much current|
