This is the first circuit anyone builds, an LED and a resistor in series, and the resistor is what stops the LED destroying itself.

The circuit

```
  ESP32 GPIO2 ───[220Ω]───▶|─── GND
                           LED
```

|From|To|Note|
|---|---|---|
|ESP32 GPIO2|Resistor leg 1|any output-capable pin|
|Resistor leg 2|LED anode (long leg)|220Ω limits current|
|LED cathode (short leg)|ESP32 GND|completes the circuit|

Three components, three connections. Build it.

Why the resistor is not optional

An LED is a diode, and a diode's current rises almost vertically once you pass its forward voltage. There is no "settling point". A red LED drops about 2.0V, and above that its resistance is very nearly zero.

So if you connect a red LED directly across a 3.3V pin:

```
Voltage the LED cannot absorb = 3.3 - 2.0 = 1.3V
Resistance in the circuit     = essentially just wire, say 1Ω
Current                       = 1.3 / 1 = 1300mA
```

1.3 amps through a 20mA part, out of a pin rated for 40mA absolute maximum. In practice the pin's own internal resistance limits it to somewhere around 100mA, which is still five times its rating.

The resistor gives that 1.3V somewhere to go.

The calculation

```
R = (Vsupply - Vf) / I
```

Worked, for a red LED on a 3.3V ESP32 pin at 10mA:

```
Vsupply = 3.3V
Vf      = 2.0V      (red, from the datasheet or just assume it)
I       = 0.010A

R = (3.3 - 2.0) / 0.010
  = 1.3 / 0.010
  = 130Ω
```

130Ω does not exist as a stock value. Round **up** to 150Ω or 220Ω. Rounding up reduces current, which is always safe. Rounding down increases it.

With the 220Ω you actually own:

```
I = (3.3 - 2.0) / 220
  = 1.3 / 220
  = 0.0059A
  = 5.9mA
```

5.9mA. That is well within the pin's ability and plenty bright indoors. Modern LEDs do not need 20mA.

Check the resistor's power rating while you are here:

```
P = I² × R
  = 0.0059² × 220
  = 0.0077W
```

Standard resistors are 0.25W, so you have 30× headroom. Not a concern here, but worth doing the check when the numbers get bigger.

Values for other colours

|Supply|LED|Vf|Resistor|Resulting current|
|---|---|---|---|---|
|3.3V|Red|2.0V|220Ω|5.9mA|
|3.3V|Red|2.0V|150Ω|8.7mA|
|3.3V|Green|2.2V|220Ω|5.0mA|
|3.3V|Yellow|2.1V|220Ω|5.5mA|
|3.3V|Blue|3.2V|**too little headroom**|use 5V instead|
|5V|Red|2.0V|330Ω|9.1mA|
|5V|Blue / white|3.2V|220Ω|8.2mA|

Blue and white on 3.3V is the interesting failure. With a Vf of 3.2V there is only 0.1V left for the resistor, so the LED is dim, flickery and wildly sensitive to the exact part. Use red or green for 3.3V GPIO work, or drive blue from the 5V pin through a transistor.

Does the resistor go before or after the LED?

Either. This is a series loop and current is identical at every point in a series loop, so the resistor limits the current whether it sits between the pin and the LED or between the LED and ground.

```
  GPIO2 ───[220Ω]───▶|─── GND       works

  GPIO2 ───▶|───[220Ω]─── GND       works identically
```

People argue about this. There is no electrical difference.

The other way to wire it, sinking

You can also connect the LED to 3V3 and have the GPIO pull it down to ground. The pin is then "sinking" current rather than "sourcing" it.

```
  3V3 ───[220Ω]───▶|─── ESP32 GPIO2
```

|From|To|Note|
|---|---|---|
|ESP32 3V3|Resistor leg 1|-|
|Resistor leg 2|LED anode (long leg)|-|
|LED cathode (short leg)|ESP32 GPIO2|-|

The logic is now inverted: `digitalWrite(2, LOW)` turns the LED **on**. This looks backwards but is common in real designs, because many chips can sink more current than they can source. On an ESP32 either is fine. Mentioning it because you will see it in other people's circuits and wonder why their code is upside down.

Code

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

Blinking without blocking, because `delay()` freezes everything and you will need this the moment you add a button:

```cpp
#define LED_PIN 2

unsigned long lastToggle = 0;
const unsigned long INTERVAL = 500;
bool ledState = false;

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  unsigned long now = millis();

  if (now - lastToggle >= INTERVAL) {
    lastToggle = now;
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
  }

  // other work can happen here without being blocked
}
```

Several LEDs

Each LED needs its **own** resistor. Do not put several LEDs in parallel behind one shared resistor.

```
   WRONG                          RIGHT

   GPIO ──[220Ω]──┬──▶|── GND     GPIO ──┬──[220Ω]──▶|── GND
                  │                      │
                  ├──▶|── GND            ├──[220Ω]──▶|── GND
                  │                      │
                  └──▶|── GND            └──[220Ω]──▶|── GND
```

LEDs do not share current evenly. Whichever has the slightly lowest forward voltage hogs most of it, so one is bright, one is dim, and the bright one dies first. Manufacturing variation guarantees they are never identical. One resistor per LED, always.

Also watch the pin total. Three LEDs at 6mA from one pin is 18mA, right at the sensible limit. More than that, use a transistor. See [[Transistor as a switch]].

The mistake everyone makes

Skipping the resistor for a quick test. It lights, it looks fine, and you have overdriven both the LED and the GPIO's output driver. Sometimes it fails immediately and sometimes it fails in a fortnight, but there is no version where nothing was damaged.

Second: LED in backwards. It does nothing at all. No light, no heat, no clue. If a fresh LED will not light, flip it before touching the code.

Third: assuming the code is wrong when GPIO2 already has the on-board LED on it. Your built-in LED may be blinking happily while the breadboard one sits dead due to a loose wire.
