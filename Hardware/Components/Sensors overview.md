A sensor turns something physical, distance or temperature or rotation, into an electrical signal your ESP32 can read.

The only questions that matter when you pick one are: what does it measure, how does it talk, and does it run on 3.3V.

The interfaces you will meet

Before the sensor table, this is the vocabulary. Every sensor uses one of these.

|Interface|Wires|How it works|Pins used|
|---|---|---|---|
|Digital|1|The pin is HIGH or LOW|Any GPIO|
|Analogue|1|A voltage between 0 and 3.3V, read with the ADC|GPIO32-39 preferred|
|PWM / pulse timing|1-2|Information is in how long a pulse lasts|Any GPIO|
|I2C|2|Shared bus, SDA + SCL, each device has an address|GPIO21 (SDA), GPIO22 (SCL)|
|SPI|4|Faster shared bus, MOSI MISO SCK + one CS per device|GPIO23/19/18 + any CS|
|UART|2|Serial, TX and RX crossed over|GPIO16/17|
|1-Wire|1|Data and sometimes power on one line|Any GPIO, needs 4.7kΩ pull-up|

I2C is the one you want most of the time. Two wires can carry a dozen sensors, and pins are the scarce resource on an ESP32.

Common sensors

|Sensor|Measures|Interface|3V3 safe?|Typical part|Use for|
|---|---|---|---|---|---|
|Ultrasonic|Distance, 2cm - 400cm|Pulse timing, 2 pins|**No, 5V echo**|HC-SR04|Obstacle avoidance, tank level|
|Ultrasonic (3V3)|Distance|Pulse timing|Yes|HC-SR04P, RCWL-1601|Same, without the level shifting|
|IMU|Acceleration + rotation|I2C|Yes|MPU6050, MPU9250|Tilt, orientation, gesture, balance robots|
|Temp + humidity|°C and %RH|1-Wire-ish digital|Yes|DHT22, DHT11|Room monitoring, weather station|
|Temp + humidity + pressure|°C, %RH, hPa|I2C|Yes|BME280|Weather station, altitude|
|Temperature only|°C, high accuracy|1-Wire|Yes|DS18B20|Waterproof probe, multiple on one wire|
|IR obstacle|Something is close|Digital|Yes|FC-51|Line following, cliff detection|
|IR receiver|Remote control codes|Digital|Yes|TSOP38238, VS1838B|Reading a TV remote|
|PIR motion|A warm body moved|Digital|Usually|HC-SR501|Motion activated lights|
|Hall effect|Magnetic field present|Digital or analogue|Yes|A3144, SS49E|RPM counting, door open/closed, position|
|Rotary encoder|Rotation, direction and amount|2 digital pins|Yes|KY-040, motor encoders|Volume knobs, wheel odometry|
|Light level|Brightness|Analogue|Yes|LDR + resistor|Day/night detection|
|Light (accurate)|Lux|I2C|Yes|BH1750, TSL2561|Proper light measurement|
|Current|Amps through a wire|Analogue|**Check, some are 5V**|ACS712, INA219 (I2C)|Battery monitoring, motor load|
|Soil moisture|Wetness|Analogue|Yes|Capacitive type|Plant watering|
|Gas / air quality|CO2, VOC|I2C or analogue|Varies|MQ-series, SGP30|Air monitoring|
|Load cell|Weight|Needs HX711 amp|Yes|HX711 + cell|Scales|
|Camera|Images|Parallel/dedicated|Yes|OV2640 on ESP32-CAM|Vision|
|GPS|Position|UART|Yes|NEO-6M|Location tracking|
|RFID|Card ID|SPI|Yes|RC522|Access control|

The 3.3V column is the one to actually read

The ESP32 is a 3.3V chip and its GPIO pins are **not** 5V tolerant. Putting 5V on a pin damages it, sometimes immediately, sometimes slowly.

The classic offender is the **HC-SR04 ultrasonic sensor**. It needs 5V to work properly, and its `ECHO` output is a 5V signal going straight into your ESP32. Two fixes:

|Fix|How|
|---|---|
|Voltage divider on ECHO|1kΩ and 2kΩ resistors, drops 5V to 3.3V|
|Buy the HC-SR04**P**|Runs entirely on 3.3V, costs the same|

The divider version:

```
   HC-SR04 ECHO ───[1kΩ]───┬───[2kΩ]─── GND
                           │
                     ESP32 GPIO18
```

|From|To|Note|
|---|---|---|
|HC-SR04 VCC|5V pin|it needs 5V to transmit|
|HC-SR04 GND|ESP32 GND|-|
|ESP32 GPIO5|HC-SR04 TRIG|3.3V out is read fine as HIGH|
|HC-SR04 ECHO|1kΩ resistor|-|
|1kΩ resistor|ESP32 GPIO18 and 2kΩ resistor|junction sits at 3.3V|
|2kΩ resistor|GND|-|

Full explanation in [[Voltage dividers]] and [[Level shifting 3V3 and 5V]].

Reading an ultrasonic sensor

The interface is unusual and worth understanding once. You send a 10µs pulse on TRIG, the sensor chirps, then it holds ECHO high for exactly as long as the sound took to come back.

```cpp
#define TRIG 5
#define ECHO 18

void setup() {
  Serial.begin(115200);
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
}

float readDistanceCm() {
  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG, LOW);

  // returns 0 on timeout after 30ms (~5m)
  long us = pulseIn(ECHO, HIGH, 30000);
  if (us == 0) return -1;

  // sound is ~343 m/s = 29.1us per cm, and it travels there AND back
  return us / 58.0;
}

void loop() {
  float d = readDistanceCm();
  if (d < 0) Serial.println("out of range");
  else       Serial.println(d);
  delay(100);
}
```

Reading an I2C sensor

I2C is why you should prefer it. Two wires, and adding a second sensor means adding no wires at all.

```
   ESP32 3V3  ──┬────────── SENSOR VCC
                │
   ESP32 GPIO21 ┼────────── SENSOR SDA
   ESP32 GPIO22 ┼────────── SENSOR SCL
                │
   ESP32 GND  ──┴────────── SENSOR GND
```

|From|To|Note|
|---|---|---|
|ESP32 3V3|Sensor VCC|breakout boards usually have pull-ups fitted|
|ESP32 GPIO21|Sensor SDA|data|
|ESP32 GPIO22|Sensor SCL|clock|
|ESP32 GND|Sensor GND|-|
|Second sensor|Same four wires|as long as the addresses differ|

Scanner sketch, the first thing to run when an I2C sensor "does not work":

```cpp
#include <Wire.h>

void setup() {
  Serial.begin(115200);
  Wire.begin(21, 22);          // SDA, SCL
  Serial.println("scanning...");

  for (byte addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.print("found device at 0x");
      Serial.println(addr, HEX);
    }
  }
  Serial.println("done");
}

void loop() {}
```

If the scan finds nothing, the problem is wiring or power, not your sensor library. If it finds the device, the wiring is proven good and you can debug the code.

The ESP32 ADC, and its annoyances

Analogue sensors go through the ADC, and the ESP32's is famously mediocre.

|Thing|Detail|
|---|---|
|Resolution|12 bit, so 0 - 4095|
|Range|0 to 3.3V with default attenuation set to 11dB|
|Linearity|Poor at the extremes, unreliable below ~0.1V and above ~3.1V|
|ADC2 pins|**Unusable while WiFi is on**. Use ADC1 only|
|ADC1 pins|GPIO32, 33, 34, 35, 36, 39|

Use ADC1 pins for anything analogue. If your analogue reading works perfectly until you connect to WiFi and then returns garbage, you are on an ADC2 pin.

```cpp
#define LDR_PIN 34    // ADC1, input only, safe with WiFi

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);        // 0 - 4095
}

void loop() {
  int raw = analogRead(LDR_PIN);
  float volts = raw * (3.3 / 4095.0);
  Serial.print(raw);
  Serial.print("  ");
  Serial.println(volts);
  delay(200);
}
```

Rotary encoders, briefly

An encoder gives two square waves 90° out of phase. Which one leads tells you direction, and counting the edges tells you how far.

```
   A  ──┐  ┌──┐  ┌──┐  ┌──
        └──┘  └──┘  └──┘
   B  ────┐  ┌──┐  ┌──┐  ┌
          └──┘  └──┘  └──

   A leads B  ->  clockwise
   B leads A  ->  anticlockwise
```

|From|To|Note|
|---|---|---|
|Encoder VCC|ESP32 3V3|-|
|Encoder GND|ESP32 GND|-|
|Encoder CLK / A|ESP32 GPIO25|use an interrupt|
|Encoder DT / B|ESP32 GPIO26|read this inside the interrupt|
|Encoder SW|ESP32 GPIO27|the push switch, needs INPUT_PULLUP|

Read encoders with interrupts, not polling. If you poll in `loop()` you will miss pulses whenever anything else takes time.

Picking a sensor

|Question|Answer|
|---|---|
|Does it run on 3.3V?|If not, you need [[Level shifting 3V3 and 5V]]|
|Does a well-maintained library exist?|Check the Arduino library manager first, this saves days|
|How many pins does it eat?|I2C costs you two shared pins, SPI costs four plus one each|
|How much current?|Most sensors are under 20mA and can run from the `3V3` pin|
|Do I need accuracy or just a threshold?|A £1 IR sensor beats a £30 lidar for "is there a wall"|

The mistake everyone makes

Feeding a 5V sensor output straight into a GPIO. The HC-SR04's ECHO pin is the usual culprit. It often appears to work, which is the worst outcome, because the pin degrades over weeks and then fails for no visible reason.

Second: using an ADC2 pin for an analogue sensor and then turning on WiFi. Reading goes dead or random. Move to GPIO32-39.

Third: debugging library code when the wiring is wrong. Run the I2C scanner first. If the address does not appear, no amount of code changes will help.

Fourth: powering a hungry sensor from the `3V3` pin alongside everything else and browning out the board.
