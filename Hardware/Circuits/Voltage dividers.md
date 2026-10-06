Two resistors in series split a voltage between them in proportion to their values, and tapping the point in the middle gives you a smaller copy of the input.

This is how you read a 9V battery on a pin that dies above 3.3V.

The circuit

```
   Vin ───[R1]───┬───[R2]─── GND
                 │
               Vout
```

|From|To|Note|
|---|---|---|
|Input voltage|R1 leg 1|the higher voltage you want to measure|
|R1 leg 2|R2 leg 1|this junction is the output|
|R1/R2 junction|ESP32 ADC pin|use an ADC1 pin: GPIO32-39|
|R2 leg 2|GND|-|
|Input GND|ESP32 GND|**shared ground or the reading is meaningless**|

The formula

```
Vout = Vin × R2 / (R1 + R2)
```

The bottom resistor over the total. That is it.

Worked example

Reading a 9V battery safely on a 3.3V ADC. You need to scale 9V down to at most 3.3V, so at minimum a divide by 2.7. Give yourself margin, aim for divide by 4, so a fresh 9.6V battery reads about 2.4V and there is headroom.

Pick R1 = 30kΩ, R2 = 10kΩ:

```
Vout = 9 × 10000 / (30000 + 10000)
     = 9 × 10000 / 40000
     = 9 × 0.25
     = 2.25V
```

2.25V, comfortably inside the ADC's range. And a fully fresh 9.6V battery gives 2.4V, still safe.

30kΩ is not an E12 value. Three 10kΩ in series works, or use R1 = 33kΩ which is stock:

```
Vout = 9 × 10000 / 43000
     = 2.09V
```

Fine. Then in code you multiply back by the ratio to recover the real voltage.

Standard useful ratios

|R1|R2|Divides by|Max input for 3.3V out|
|---|---|---|---|
|10kΩ|10kΩ|2.0|6.6V|
|10kΩ|4.7kΩ|3.13|10.3V|
|20kΩ|10kΩ|3.0|9.9V|
|33kΩ|10kΩ|4.3|14.2V|
|47kΩ|10kΩ|5.7|18.8V|
|100kΩ|10kΩ|11.0|36.3V|
|1kΩ|2kΩ|1.5|4.95V, this is the HC-SR04 one|

That last row is the common 5V-to-3.3V signal divider. R1 = 1kΩ, R2 = 2kΩ gives `5 × 2/3 = 3.33V`, which is right on the limit. Use 1kΩ and 2kΩ, or safer, 2.2kΩ and 3.3kΩ giving 3.0V.

Picking resistor sizes, not just the ratio

The ratio sets the output voltage but the absolute values matter for two other reasons.

Too small and you waste power. 100Ω and 100Ω across a 9V battery draws 45mA continuously, which flattens the battery in a day for no reason.

Too large and the ADC misreads. The ESP32's ADC input has some leakage and its own capacitance to charge, and with very high resistances the divider cannot supply enough current to settle. Above about 100kΩ total, readings drift.

The sweet spot is a total of 10kΩ to 100kΩ. At 40kΩ total on 9V you draw 0.22mA, which is negligible but plenty to drive the ADC.

```
Current through the divider = Vin / (R1 + R2)
                            = 9 / 40000
                            = 0.000225A
                            = 0.22mA
```

Reading it on the ESP32

```cpp
#define BATT_PIN 34        // ADC1, input only, works with WiFi

const float R1 = 33000.0;
const float R2 = 10000.0;
const float RATIO = (R1 + R2) / R2;    // 4.3

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);            // 0 - 4095
  analogSetAttenuation(ADC_11db);      // full 0 - 3.3V range
}

float readBatteryVolts() {
  // average several readings, the ESP32 ADC is noisy
  long sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += analogRead(BATT_PIN);
    delay(2);
  }
  float raw = sum / 16.0;

  float pinVolts = raw * (3.3 / 4095.0);
  return pinVolts * RATIO;
}

void loop() {
  Serial.print("battery: ");
  Serial.print(readBatteryVolts(), 2);
  Serial.println("V");
  delay(1000);
}
```

Averaging matters. A single `analogRead()` on an ESP32 jumps around by tens of counts. Sixteen samples averaged is much steadier.

Calibrating

Your reading will be a bit off, for three reasons: 5% resistor tolerance, the ADC's reference not being exactly 3.3V, and the ESP32 ADC being noticeably non-linear.

Fix it empirically. Measure the real battery voltage with a multimeter, note what your code prints, and apply a correction factor.

```cpp
const float CALIBRATION = 1.043;    // multimeter says 9.02, code said 8.65

float readBatteryVolts() {
  // ... as above ...
  return pinVolts * RATIO * CALIBRATION;
}
```

Not elegant but it is what everyone does, and it takes the error from 5% to under 1%.

Also worth knowing: the ESP32 ADC is unreliable below about 0.15V and above about 3.1V, where it flattens out and stops responding. Design your divider so the normal operating range sits between 0.5V and 2.8V, well inside the good region. That is another reason to divide by 4 rather than by exactly 2.7.

Battery voltage to state of charge

Once you have the voltage, the mapping depends on the chemistry.

|Battery|Full|Nominal|Empty, stop here|
|---|---|---|---|
|LiPo, 1 cell|4.2V|3.7V|3.0V|
|LiPo, 2 cell|8.4V|7.4V|6.0V|
|LiPo, 3 cell|12.6V|11.1V|9.0V|
|Alkaline AA|1.5V|1.5V|0.9V|
|NiMH AA|1.4V|1.2V|1.0V|
|9V alkaline|9.6V|9V|6.0V|

Voltage is a poor charge gauge under load, since the voltage sags while a motor runs and springs back when it stops. Sample it when things are idle.

What a divider cannot do

A voltage divider is not a power supply. It is fine for measuring a voltage with an ADC, because an ADC input draws almost no current. Try to power anything from the midpoint and it collapses.

If you draw current out of the middle, that current also flows through R2, changing the maths entirely. Powering a 10mA sensor from a 33kΩ/10kΩ divider gives you nothing useful, because the divider itself can only supply 0.22mA.

To actually make a lower voltage that can supply current, use a [[Voltage Regulators]].

Using it for a 5V sensor signal

A divider is the cheapest way to bring a 5V output down to 3.3V. This is exactly the HC-SR04 ECHO pin problem.

```
   HC-SR04 ECHO ───[1kΩ]───┬───[2kΩ]─── GND
                           │
                     ESP32 GPIO18
```

|From|To|Note|
|---|---|---|
|HC-SR04 ECHO|1kΩ resistor leg 1|5V logic output|
|1kΩ resistor leg 2|ESP32 GPIO18 and 2kΩ leg 1|junction sits at 3.33V|
|2kΩ resistor leg 2|ESP32 GND|-|
|Sensor GND|ESP32 GND|shared|

This works for slow signals. It does not work for fast ones like SPI or I2C clock, because the resistance plus the pin's capacitance rounds the edges off. For those, use a proper level shifter. See [[Level shifting 3V3 and 5V]].

Potentiometer as an adjustable divider

A potentiometer is a voltage divider you can turn. The wiper slides along a resistive track, changing R1 and R2 while keeping their total constant.

```
   3V3 ────┬─────
           │
          ┌┴┐
          │ │◄──── wiper ──── ESP32 GPIO34
          └┬┘  10kΩ pot
           │
   GND ────┴─────
```

|From|To|Note|
|---|---|---|
|ESP32 3V3|Pot outer leg 1|-|
|Pot wiper (middle leg)|ESP32 GPIO34|reads 0 to 3.3V as you turn it|
|Pot outer leg 2|ESP32 GND|swap the outers to reverse the direction|

```cpp
#define POT_PIN 34

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);
}

void loop() {
  int raw = analogRead(POT_PIN);        // 0 - 4095
  int pct = map(raw, 0, 4095, 0, 100);
  Serial.println(pct);
  delay(100);
}
```

The mistake everyone makes

Trying to power something from a divider. It is a measurement tool, not a supply. The moment you draw current the output voltage drops and the maths goes out of the window.

Second: using an ADC2 pin. GPIO0, 2, 4, 12-15, 25-27 are ADC2 and **stop working entirely once WiFi is enabled**. Your divider reads perfectly on the bench and returns garbage the moment you connect to a network. Use GPIO32-39.

Third: no shared ground between the thing being measured and the ESP32. A voltage is only meaningful relative to a reference, and without a common ground there is no reference.

Fourth: forgetting the divider is always drawing current. Leave a 10kΩ/10kΩ divider on a LiPo and it drains it slowly even when the project is off. Use larger values or a MOSFET to disconnect it when sleeping.
