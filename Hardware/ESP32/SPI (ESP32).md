SPI uses four wires instead of two and is far faster than I2C, which is why SD cards and displays use it.

The four wires

|Wire|Also called|Direction|What it does|
|---|---|---|---|
|SCK|CLK, SCLK|master out|The clock, one tick per bit|
|MOSI|SDI, DIN, COPI|master out|Data going to the device|
|MISO|SDO, DOUT, CIPO|master in|Data coming back|
|CS|SS, CE|master out|Chip select, picks which device is listening|

Default pins on an ESP32 (VSPI)

|Signal|GPIO|
|---|---|
|SCK|18|
|MISO|19|
|MOSI|23|
|CS|5 (but any free pin works)|

```
  three devices, one bus, three chip selects

   ESP32
   ┌──────────┐
   │ SCK  18 ├──┬────────────┬────────────┬─────
   │ MOSI 23 ├──┼──┬─────────┼──┬─────────┼──┬──
   │ MISO 19 ├──┼──┼──┬──────┼──┼──┬──────┼──┼──┬─
   │         │  │  │  │      │  │  │      │  │  │
   │ CS   5  ├──┼──┼──┼──┐   │  │  │      │  │  │
   │ CS   17 ├──┼──┼──┼──┼───┼──┼──┼──┐   │  │  │
   │ CS   16 ├──┼──┼──┼──┼───┼──┼──┼──┼───┼──┼──┼──┐
   └──────────┘  │  │  │  │   │  │  │  │   │  │  │  │
              ┌──┴──┴──┴──┴┐ ┌┴──┴──┴──┴┐ ┌┴──┴──┴──┴┐
              │ SD card    │ │ display  │ │ ADC      │
              └────────────┘ └──────────┘ └──────────┘

  Only ONE CS is pulled LOW at a time. The others ignore
  everything on the bus. CS is active LOW, always.
```

A basic transfer

```cpp
/*
1. SPI.begin() sets up the bus, CS you manage yourself
2. beginTransaction sets speed, bit order and mode for this device
3. pull CS low, transfer bytes, pull CS high again
4. transfer() sends and receives at the same time, always, that's how SPI works
*/

#include <SPI.h>

#define CS 5

void setup() {
    Serial.begin(115200);

    pinMode(CS, OUTPUT);
    digitalWrite(CS, HIGH);       // idle high = nobody selected

    SPI.begin();                  // SCK 18, MISO 19, MOSI 23
}

byte readRegister(byte reg) {
    SPI.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));

    digitalWrite(CS, LOW);
    SPI.transfer(reg | 0x80);          // top bit set = read, on many chips
    byte value = SPI.transfer(0x00);   // send a dummy byte to clock the answer out
    digitalWrite(CS, HIGH);

    SPI.endTransaction();
    return value;
}

void loop() {
    Serial.println(readRegister(0x0F), HEX);   // output: 6B   a "who am I" register
    delay(1000);
}
```

Custom pins, because the ESP32 can remap almost anything

```cpp
#include <SPI.h>

SPIClass hspi(HSPI);      // the ESP32 has two usable SPI buses: VSPI and HSPI

void setup() {
    hspi.begin(14, 12, 13, 15);   // SCK, MISO, MOSI, CS
}
```

The four SPI modes

```
  Mode is the clock polarity (idle level) and phase (which edge samples).
  The datasheet tells you which one. Wrong mode = consistent garbage.

  MODE0   CPOL 0, CPHA 0    idle low,  sample on rising     ← most common
  MODE1   CPOL 0, CPHA 1    idle low,  sample on falling
  MODE2   CPOL 1, CPHA 0    idle high, sample on falling
  MODE3   CPOL 1, CPHA 1    idle high, sample on rising     ← second most common

  MODE 0:
   SCK  ──┐  ┌──┐  ┌──┐  ┌──
          └──┘  └──┘  └──┘
          ▲     ▲     ▲        data sampled here
   MOSI ──╳═════╳═════╳═════
          b7    b6    b5
```

Writing to an SD card, the everyday use

```cpp
/*
1. the SD library handles the SPI protocol for you, you just give it the CS pin
2. always close the file, otherwise the data never actually lands on the card
3. cards must be FAT32 formatted
*/

#include <SPI.h>
#include <SD.h>

#define SD_CS 5

void setup() {
    Serial.begin(115200);

    if (!SD.begin(SD_CS)) {
        Serial.println("card failed");     // wiring, format, or a 5V-only module
        return;
    }

    File f = SD.open("/log.txt", FILE_APPEND);
    if (f) {
        f.println("hello from the esp32");
        f.close();                          // this is the step people forget
        Serial.println("written");
    }
}

void loop() {}

// output: written
```

SPI or I2C

|                |I2C|SPI|
|---|---|---|
|Wires|2 (+ pull-ups)|4 + one CS per device|
|Speed|100k - 400k, 1M pushing it|10 MHz easily, 80 MHz possible|
|Devices|Many, limited by addresses|Many, limited by spare pins|
|Addressing|Built in|None, CS does it|
|Acknowledgement|Yes, you know it worked|No, it just clocks bits out blindly|
|Best for|Sensors, RTCs, small OLEDs|SD cards, TFT displays, fast ADCs|

```
  Rule of thumb:
    a slow sensor and you're short of pins   -> I2C
    a display refreshing, or an SD card      -> SPI
    a device that offers both                -> I2C unless it's too slow
```

The gotcha

```
  Symptom: reads always come back 0x00 or 0xFF.

    0xFF  -> MISO is floating, device not selected or not powered
    0x00  -> MISO shorted low, or the wrong SPI mode

  CS must go HIGH between transactions. Leaving it low keeps the
  device listening and the next command runs into the previous one.

  And SD card modules are often 5V. The ones with an onboard
  regulator and level shifter are the safe ones for a 3.3V ESP32.
```
