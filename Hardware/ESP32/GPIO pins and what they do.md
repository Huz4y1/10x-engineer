The ESP32 has 34 GPIO numbers printed on it and only about half are actually safe to use, this table is the thing worth memorising.

The full pin table

|GPIO|Safe to use|Input|Output|Notes|
|---|---|---|---|---|
|0|careful|pull-up|yes|Strapping pin. Low at boot = flash mode. Has a pull-up|
|1|no|TX0|TX0|Serial monitor TX, using it breaks uploads|
|2|careful|yes|yes|Strapping pin, onboard LED. Must not be held high at boot|
|3|no|RX0|RX0|Serial monitor RX, high at boot|
|4|yes|yes|yes|Good general pin, ADC2 and touch|
|5|careful|yes|yes|Strapping pin, outputs a PWM burst at boot. Has a pull-up|
|6|NEVER|-|-|Flash SCK|
|7|NEVER|-|-|Flash SDO|
|8|NEVER|-|-|Flash SDI|
|9|NEVER|-|-|Flash SHD|
|10|NEVER|-|-|Flash SWP|
|11|NEVER|-|-|Flash CSC|
|12|careful|yes|yes|Strapping pin. HIGH at boot = wrong flash voltage = no boot|
|13|yes|yes|yes|Good general pin|
|14|yes|yes|yes|Outputs a PWM burst at boot|
|15|careful|yes|yes|Strapping pin, silences boot log if low. Has a pull-up|
|16|yes|yes|yes|Good general pin (not on WROVER, used by PSRAM)|
|17|yes|yes|yes|Good general pin (not on WROVER)|
|18|yes|yes|yes|Default SPI SCK|
|19|yes|yes|yes|Default SPI MISO|
|21|yes|yes|yes|Default I2C SDA|
|22|yes|yes|yes|Default I2C SCL|
|23|yes|yes|yes|Default SPI MOSI|
|25|yes|yes|yes|DAC1, real analogue output|
|26|yes|yes|yes|DAC2, real analogue output|
|27|yes|yes|yes|Good general pin|
|32|yes|yes|yes|ADC1, works with WiFi on. Best analogue pin|
|33|yes|yes|yes|ADC1, works with WiFi on|
|34|input only|yes|NO|No output driver, no internal pull-up|
|35|input only|yes|NO|No output driver, no internal pull-up|
|36 (VP)|input only|yes|NO|No output driver, no internal pull-up|
|39 (VN)|input only|yes|NO|No output driver, no internal pull-up|

GPIO 20, 24, 28-31 are not brought out on a standard DevKit. 37 and 38 do not exist on WROOM.

The four groups, said plainly

```
  ┌─────────────────────────────────────────────────────────┐
  │ 6 7 8 9 10 11   flash          NEVER TOUCH              │
  │                                soldered to the flash    │
  │                                chip, using them bricks  │
  │                                the running program      │
  ├─────────────────────────────────────────────────────────┤
  │ 0 2 5 12 15     strapping      read at boot to decide   │
  │                                how to start. Wire       │
  │                                something that holds     │
  │                                them and it won't boot   │
  ├─────────────────────────────────────────────────────────┤
  │ 34 35 36 39     input only     no output transistor at  │
  │                                all, and NO internal     │
  │                                pull-up. External        │
  │                                resistor required        │
  ├─────────────────────────────────────────────────────────┤
  │ the rest        free           use these first          │
  └─────────────────────────────────────────────────────────┘
```

Pins to reach for first, in order

|Need|Use|
|---|---|
|LED / relay / output|13, 14, 27, 26, 25|
|Button|4, 13, 27 with INPUT_PULLUP|
|Analogue in|32, 33 (ADC1, survives WiFi)|
|I2C|21 SDA, 22 SCL|
|SPI|18 SCK, 19 MISO, 23 MOSI, 5 CS|
|Real voltage out|25, 26 (DAC)|
|Deep sleep wake|32, 33, or any RTC pin|

What a strapping pin failure looks like

```cpp
/*
1. GPIO 12 is read at boot to set the flash voltage
2. if it is HIGH at boot the chip picks 1.8V, the flash needs 3.3V
3. result: the board never starts, or boot loops forever
4. so a button pulling 12 HIGH, or an LED wired 3V3 -> 12, breaks the board
*/

// bad
pinMode(12, INPUT_PULLUP);   // now 12 is high at boot, board may not start

// fine
pinMode(13, INPUT_PULLUP);   // 13 is not a strapping pin
```

Input-only pins need their own resistor

```cpp
/*
1. 34-39 have no internal pull-up, INPUT_PULLUP silently does nothing
2. the pin floats and reads random noise
3. so add a real 10k resistor on the board
*/

void setup() {
    Serial.begin(115200);
    pinMode(34, INPUT);          // INPUT_PULLUP here would be a lie
}

void loop() {
    Serial.println(digitalRead(34));
    delay(200);
}
```

```
  external pull-up for an input-only pin

     3V3
      │
     ┌┴┐
     │ │ 10k
     └┬┘
      ├──────── GPIO 34      reads HIGH when the button is open
      │
     ─┴─  button
      │
     GND                     reads LOW when pressed
```

The 3.3V rule again, because it destroys boards

```
  Every pin here is 3.3V logic and NOT 5V tolerant.

  5V sensor output ──X──> GPIO        kills the pin
  5V sensor output ──[level shifter]──> GPIO      fine

  Quick divider for a 5V signal into a GPIO:

    5V signal ──┬── 10k ──┬── 20k ── GND
                          │
                          └──> GPIO   ≈ 3.3V

  Current per pin: about 12mA comfortably, 40mA absolute max.
  An LED needs a resistor. A motor needs a transistor, never a pin.
```
