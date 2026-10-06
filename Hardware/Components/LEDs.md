An LED is a diode that gives off light when current flows through it the right way, and it has no ability to limit that current itself.

That last part is the whole reason you need a resistor. An LED is not a light bulb. A bulb has resistance and settles at a sensible current. An LED behaves more like a switch that snaps open below a certain voltage and becomes almost a dead short above it. Give it a voltage and it will pull as much current as the supply can deliver, get very bright for a moment, and die.

Schematic symbol

```
  3V3 ───[220Ω]───▶|─── GND
                   LED
```

The triangle points in the direction current flows, and the bar is the blocking side. Current goes anode → cathode, left to right here.

Which leg is which

|Clue|Anode (+)|Cathode (-)|
|---|---|---|
|Leg length|Long|Short|
|Case edge|Round|Flat spot on the rim|
|Inside the LED|Small pointy post|Big cup shaped bit|

The flat spot is the reliable one, because legs get trimmed. Big cup = negative.

Forward voltage

Every LED drops a fixed-ish voltage across itself once it is conducting. This depends on the colour, because colour is set by the semiconductor chemistry.

|Colour|Forward voltage (Vf)|Typical current|
|---|---|---|
|Red|1.8V - 2.0V|20mA|
|Yellow / amber|2.0V - 2.2V|20mA|
|Green|2.0V - 3.0V|20mA|
|Blue|3.0V - 3.4V|20mA|
|White|3.0V - 3.4V|20mA|
|IR (invisible)|1.2V - 1.5V|20mA to 100mA|

Note the problem hiding in that table. A blue or white LED can have a forward voltage of 3.3V, which is exactly the ESP32's supply. There is nothing left over for the resistor, so it will be dim or not light at all. Run blue and white LEDs from the 5V pin (VIN / V5 on most dev boards) with a bigger resistor, still switched by a transistor or driven from a 5V-tolerant path. Red and green are the sensible choice for direct 3.3V GPIO work.

Calculating the resistor

```
R = (Vsupply - Vf) / I
```

Worked example, red LED on an ESP32 GPIO:

```
Vsupply = 3.3V   (ESP32 GPIO high)
Vf      = 2.0V   (red)
I       = 0.010A (10mA, plenty bright)

R = (3.3 - 2.0) / 0.010
R = 1.3 / 0.010
R = 130Ω
```

130Ω is not an E12 value. Round UP to the next one you own, 150Ω or 220Ω. Rounding up is always safe, it just means slightly less current and slightly less brightness. Rounding down pushes more current than you calculated.

Quick reference so you can stop doing the maths:

|Supply|LED colour|Resistor|Result|
|---|---|---|---|
|3.3V|Red / yellow|220Ω|~6mA, clearly visible|
|3.3V|Red / yellow|150Ω|~9mA, bright|
|3.3V|Green|220Ω|~6mA|
|5V|Red|330Ω|~9mA|
|5V|Blue / white|100Ω|~17mA, bright|
|5V|Blue / white|220Ω|~8mA|

Modern LEDs are far more efficient than the 20mA figure in the datasheet suggests. At 5mA a red LED is already annoyingly bright in a dark room. Do not feel you have to hit 20mA.

The ESP32 current limit

An ESP32 GPIO pin can source about 40mA absolute maximum, and 20mA is the sane working figure. One LED at 10mA is fine. Eight LEDs at 20mA each from eight pins is 160mA total and will exceed what the chip can dump through its ground pin. For more than a few LEDs, use a transistor or a dedicated driver.

Building it

```
  ESP32 GPIO2 ───[220Ω]───▶|─── GND
                           LED
```

|From|To|Note|
|---|---|---|
|ESP32 GPIO2|Resistor leg 1|any output-capable pin|
|Resistor leg 2|LED anode (long leg)|220Ω limits current|
|LED cathode (short leg)|ESP32 GND|completes the circuit|

```cpp
#define LED_PIN 2

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(500);
  digitalWrite(LED_PIN, LOW);
  delay(500);
}
```

Fading it, the ESP32 way

The ESP32 does not use `analogWrite`. It has a hardware LED controller called LEDC. In recent Arduino-ESP32 cores you attach a pin to it and write a duty cycle.

```cpp
#define LED_PIN 2

void setup() {
  // pin, frequency in Hz, resolution in bits
  ledcAttach(LED_PIN, 5000, 8);   // 8 bit -> 0 to 255
}

void loop() {
  for (int b = 0; b <= 255; b++) {
    ledcWrite(LED_PIN, b);
    delay(4);
  }
  for (int b = 255; b >= 0; b--) {
    ledcWrite(LED_PIN, b);
    delay(4);
  }
}
```

The brightness is faked by switching the LED on and off thousands of times a second and varying how long it stays on. Your eye averages it.

The mistake everyone makes

Wiring the LED straight from a pin to GND with no resistor "just to test it". It will light. It will look fine. You have just pushed somewhere north of 100mA through a 20mA part and through the GPIO's output driver. Sometimes the LED dies, sometimes the pin dies, sometimes it works for weeks and then dies. There is no version of this where you got away with it. The resistor costs a fraction of a penny.

Second mistake: LED in backwards, then concluding the code is broken. An LED wired backwards does nothing at all, no light, no heat, no clue. If a fresh LED does not light, flip it before you touch the code.

Third: forgetting the on-board LED on most ESP32 dev boards is on GPIO2, so your test blink might be working on the built-in LED while your breadboard one sits dead because of a loose wire.
