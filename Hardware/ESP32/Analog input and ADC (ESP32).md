The ADC turns a voltage on a pin into a number, so you can read a knob, a light sensor or a battery.

Reading a value

```cpp
/*
1. the ESP32 ADC is 12-bit, so 0 to 4095
2. 0 means 0V, 4095 means roughly 3.3V
3. anything above 3.3V on the pin damages it, this is not a 5V input
*/

#define POT 32

void setup() {
    Serial.begin(115200);
}

void loop() {
    int raw = analogRead(POT);
    Serial.println(raw);
    delay(200);
}

// output: 0
//         2047
//         4095
```

```
  wiring a potentiometer

    3V3 ──┬─────────┐
          │         │
         ┌┴─────────┴┐
         │    pot    │   outer legs to 3V3 and GND
         └─────┬─────┘   middle (wiper) to the ADC pin
               │
    GPIO 32 ───┘
               │
    GND ───────┘

  turning the knob slides the wiper between 0V and 3.3V
```

The two ADCs, and why it matters

|            |ADC1|ADC2|
|---|---|---|
|Pins|32, 33, 34, 35, 36, 39|0, 2, 4, 12, 13, 14, 15, 25, 26, 27|
|Works with WiFi on|yes|NO|
|Use it|always|only if WiFi is off|

```cpp
/*
1. ADC2 is shared with the WiFi radio
2. once WiFi.begin() runs, analogRead on an ADC2 pin returns 0 or garbage
3. it doesn't error, it just silently stops working, which is worse
4. so put every analogue sensor on 32/33/34/35/36/39
*/

WiFi.begin(ssid, pass);

Serial.println(analogRead(32));   // output: 2043   fine, ADC1
Serial.println(analogRead(4));    // output: 0      ADC2, dead while WiFi is up
```

Converting a reading to volts

```cpp
/*
1. reading / 4095 gives a fraction of full scale
2. multiply by the reference, 3.3V
3. float maths is fine here on an ESP32, on a tiny MCU see [[Fixed point and avoiding floats]]
*/

void loop() {
    int raw = analogRead(32);

    float volts = raw * (3.3f / 4095.0f);

    Serial.print(raw);
    Serial.print("  ");
    Serial.println(volts, 3);
    delay(500);
}

// output: 0     0.000
//         2048  1.650
//         4095  3.300
```

Attenuation, which sets the voltage range

```cpp
/*
1. the raw ADC only handles about 1.1V, an attenuator divides the input down first
2. more attenuation = wider range, slightly worse accuracy
3. 11dB is the Arduino default, which is why 3.3V ≈ 4095
*/

void setup() {
    analogSetAttenuation(ADC_11db);          // all pins
    analogSetPinAttenuation(32, ADC_11db);   // or just one pin
}
```

|Setting|Usable input range|Use for|
|---|---|---|
|`ADC_0db`|0 - 1.1V|small sensor signals|
|`ADC_2_5db`|0 - 1.5V||
|`ADC_6db`|0 - 2.2V||
|`ADC_11db`|0 - 3.3V|the default, and what you normally want|

Changing the resolution

```cpp
analogReadResolution(12);   // 0-4095, the default
analogReadResolution(10);   // 0-1023, matches Arduino Uno code you copied
```

Reading a voltage higher than 3.3V

```
  A divider first. Reading a 12V battery:

   12V ──┬── 100k ──┬── 33k ── GND
                    │
                    └──> GPIO 32     12 * 33/(100+33) ≈ 2.98V, safe

  Then multiply the result back up in code:
     float vbat = volts * (133.0f / 33.0f);
```

Smoothing, because the reading jitters

```cpp
/*
1. the ESP32 ADC is noisy, a still pot still wobbles by 20-40 counts
2. averaging several reads flattens it
3. cheap and almost always enough
*/

int readSmooth(int pin) {
    long total = 0;
    for (int i = 0; i < 16; i++) {
        total += analogRead(pin);
    }
    return total / 16;
}
```

The gotcha

```
  The ESP32 ADC is genuinely non-linear, especially below ~0.15V and
  above ~3.1V. It flatlines at both ends. If you need real accuracy,
  either calibrate it yourself or use an external I2C ADC like an ADS1115.

  Also: analogRead on an INPUT_PULLUP pin reads nonsense, the pull-up
  fights the sensor. Use plain INPUT, or nothing at all.
```
