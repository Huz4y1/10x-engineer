I2C is a two-wire bus where one master talks to many sensors, each identified by an address.

The two wires

```
       3V3 ──┬──────┬─────────────────────────────
             │      │
            ┌┴┐    ┌┴┐   4.7k pull-ups, ONE pair
            │ │    │ │   for the whole bus
            └┬┘    └┬┘
             │      │
  SDA ───────┴──────┼────┬──────────┬──────────
  SCL ──────────────┴────┼───┬──────┼───┬──────
                         │   │      │   │
                    ┌────┴───┴─┐ ┌──┴───┴───┐
   ESP32            │ sensor   │ │ OLED     │
   SDA 21 ──────────│ 0x76     │ │ 0x3C     │
   SCL 22 ──────────│          │ │          │
   GND ─────────────┴──────────┴─┴──────────┘

  SDA = data, SCL = clock. Every device sits on the same
  two wires. GND must be shared.
```

Scanning the bus, always do this first

```cpp
/*
1. try to start a transmission to every address from 1 to 126
2. endTransmission() returns 0 if a device acknowledged
3. if nothing is found: wiring, pull-ups, or power
*/

#include <Wire.h>

void setup() {
    Serial.begin(115200);
    Wire.begin(21, 22);          // SDA, SCL. Defaults are 21/22 anyway

    Serial.println("scanning...");

    for (byte addr = 1; addr < 127; addr++) {
        Wire.beginTransmission(addr);
        byte error = Wire.endTransmission();

        if (error == 0) {
            Serial.print("found device at 0x");
            Serial.println(addr, HEX);
        }
    }
    Serial.println("done");
}

void loop() {}

// output: scanning...
//         found device at 0x3C
//         found device at 0x76
//         done
```

Addresses you will keep meeting

|Address|Device|
|---|---|
|0x27 / 0x3F|16x2 LCD backpack|
|0x3C / 0x3D|SSD1306 OLED|
|0x40|INA219 current sensor, SHT31|
|0x48|ADS1115 external ADC|
|0x57|MAX30102 pulse sensor|
|0x68|MPU6050 accelerometer, DS3231 RTC|
|0x76 / 0x77|BMP280 / BME280|

Writing to a device

```cpp
/*
1. beginTransmission opens a conversation with one address
2. write() queues bytes, nothing goes out yet
3. endTransmission actually sends them and returns a status
*/

#define MPU 0x68

void wakeSensor() {
    Wire.beginTransmission(MPU);
    Wire.write(0x6B);            // register number: power management
    Wire.write(0x00);            // value: 0 = wake up
    byte err = Wire.endTransmission();

    Serial.println(err);         // output: 0    0 means it acknowledged
}
```

Reading from a device

```cpp
/*
1. first write the register number you want to read from
2. endTransmission(false) keeps the bus held, a "repeated start"
3. then requestFrom asks for N bytes back
4. most sensors send 16-bit values as high byte then low byte
*/

int16_t readAxis(byte reg) {
    Wire.beginTransmission(MPU);
    Wire.write(reg);
    Wire.endTransmission(false);      // false = don't release the bus

    Wire.requestFrom(MPU, 2);

    int16_t value = (Wire.read() << 8) | Wire.read();
    return value;
}

void loop() {
    Serial.println(readAxis(0x3B));   // output: -1024
    delay(200);
}
```

Error codes from endTransmission

|Code|Meaning|
|---|---|
|0|Success|
|1|Data too long for the buffer|
|2|Address sent, got NACK — wrong address, or device not powered|
|3|Data sent, got NACK|
|4|Other error|
|5|Timeout — usually SDA or SCL stuck low, check the pull-ups|

Pull-up resistors

```
  I2C devices can only pull the line LOW. Nothing drives it HIGH,
  the resistors do. No pull-ups = the lines never return HIGH = the
  scanner finds nothing.

  4.7k     safe default
  10k      fine for a short bus, 2 devices
  2.2k     needed for a long bus or 400 kHz

  Most breakout boards already have pull-ups on them. Five boards
  each with 10k gives 2k total, which is too strong. If a long
  chain misbehaves, desolder the pull-ups on all but one board.
```

Speed and the second bus

```cpp
void setup() {
    Wire.begin(21, 22, 400000);      // 400 kHz fast mode, default is 100 kHz

    // the ESP32 has TWO I2C controllers, useful for address clashes
    Wire1.begin(33, 32);             // a second bus on different pins
}
```

The ESP-IDF way

```c
// ESP-IDF, explicit config and no hidden global object

#include "driver/i2c.h"

i2c_config_t conf = {
    .mode             = I2C_MODE_MASTER,
    .sda_io_num       = 21,
    .scl_io_num       = 22,
    .sda_pullup_en    = GPIO_PULLUP_ENABLE,   // internal pull-ups are weak,
    .scl_pullup_en    = GPIO_PULLUP_ENABLE,   // still fit real 4.7k resistors
    .master.clk_speed = 100000,
};
i2c_param_config(I2C_NUM_0, &conf);
i2c_driver_install(I2C_NUM_0, conf.mode, 0, 0, 0);
```

The gotcha

```
  Two devices with the same fixed address cannot share a bus. Options:

    - many boards have an ADDR pin or a solder jumper, tie it to
      change 0x76 into 0x77
    - put the second one on Wire1
    - use a TCA9548A I2C multiplexer

  And 3.3V. A 5V I2C module pulls SDA to 5V and feeds it straight
  into the ESP32. Use 3.3V modules, or a proper I2C level shifter.

  If you need speed rather than convenience, see [[SPI (ESP32)]].
```
