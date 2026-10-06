A button on its own is not enough, because a switch can only connect a pin to something, it cannot disconnect it to a known value.

That gap is the whole problem, and a single resistor fixes it.

The floating pin problem

Wire a button between 3V3 and GPIO4 with nothing else. Press it, the pin is at 3.3V, reads HIGH, fine. Release it, and the pin is connected to... nothing. Not ground, not 3.3V. Just an open wire.

An input pin is extremely high impedance, essentially a tiny capacitor watching for voltage. With nothing driving it, it holds whatever charge happens to be there and picks up interference from mains hum, nearby wires, and your hand moving near the board. It will read HIGH sometimes and LOW others, and it will change if you touch the wire.

```
   3V3 ───[BTN]───── ESP32 GPIO4      released = floating = random
```

This is the classic "my button works when I touch the wire" symptom. The pin is acting as an aerial and your body is the signal source.

The fix is a resistor that gently holds the pin at a known level whenever the button is not doing anything. It has to be weak enough that the button can easily override it, hence 10kΩ.

Pull-up

Resistor to 3V3, button to GND. Idle HIGH, pressed LOW.

```
                   3V3
                    │
                 [10kΩ]
                    │
                    ├───────── ESP32 GPIO4
                    │
                  [BTN]
                    │
                   GND
```

|From|To|Note|
|---|---|---|
|ESP32 3V3|Resistor leg 1|-|
|Resistor leg 2|ESP32 GPIO4|and to button leg 1, all one node|
|Button leg 1|Same node as above|-|
|Button leg 2|ESP32 GND|opposite pair on a tactile switch|

Reasoning, node by node:

|State|Path|Pin sees|Current wasted|
|---|---|---|---|
|Released|3V3 through 10kΩ to pin, dead end|3.3V, HIGH|None, no current flows|
|Pressed|3V3 through 10kΩ to GND|0V, LOW|3.3/10000 = 0.33mA|

When released there is no complete circuit, so no current flows through the resistor, so no voltage is dropped across it, so the pin sits at the full 3.3V. That is the bit that feels counterintuitive until you have thought it through once. A resistor with no current through it drops no voltage.

Pull-down

Resistor to GND, button to 3V3. Idle LOW, pressed HIGH.

```
                   3V3
                    │
                  [BTN]
                    │
                    ├───────── ESP32 GPIO4
                    │
                 [10kΩ]
                    │
                   GND
```

|From|To|Note|
|---|---|---|
|ESP32 3V3|Button leg 1|-|
|Button leg 2|ESP32 GPIO4|and to resistor leg 1|
|Resistor leg 1|Same node as above|-|
|Resistor leg 2|ESP32 GND|-|

|State|Pin sees|
|---|---|
|Released|0V, LOW|
|Pressed|3.3V, HIGH|

The logic reads more naturally, pressed = HIGH = true. So why is pull-up the standard?

Which one to use

|  |Pull-up|Pull-down|
|---|---|---|
|Idle level|HIGH|LOW|
|Pressed reads|LOW, inverted|HIGH, natural|
|Built into the ESP32|**Yes**|Yes, but less reliable on some pins|
|External parts needed|None, use INPUT_PULLUP|A 10kΩ resistor|
|Noise immunity|Better|Worse|
|Long wire runs|Preferred|Avoid|
|Industry default|**Yes**|Rare|

Use pull-up. The inverted logic is a minor annoyance you get used to in a day, and you get it for free with no components.

The noise argument is real. Electrical interference tends to couple in as brief positive spikes. On a pull-down circuit an idle-LOW pin can get spiked HIGH and register a phantom press. On a pull-up circuit the idle-HIGH pin is already at 3.3V and a positive spike does nothing.

The ESP32 internal pull-up, the version you actually build

The chip contains a resistor of roughly 45kΩ that you can switch in from software. So the real-world circuit is two wires.

```
   (internal ~45kΩ pull-up inside the ESP32)
                    ┊
   ESP32 GPIO4 ─────┴──[BTN]─── GND
```

|From|To|Note|
|---|---|---|
|ESP32 GPIO4|Button leg 1|-|
|Button leg 2|ESP32 GND|use legs from opposite sides of the switch|
|-|-|no external resistor at all|

```cpp
#define BTN_PIN 4

void setup() {
  Serial.begin(115200);
  pinMode(BTN_PIN, INPUT_PULLUP);    // resistor enabled inside the chip
}

void loop() {
  // LOW means pressed, because the button pulls the pin down to GND
  if (digitalRead(BTN_PIN) == LOW) {
    Serial.println("pressed");
  }
  delay(100);
}
```

Detecting a press rather than a hold. Almost always what you want, and it needs you to watch for the *change*:

```cpp
#define BTN_PIN 4

int lastState = HIGH;

void setup() {
  Serial.begin(115200);
  pinMode(BTN_PIN, INPUT_PULLUP);
}

void loop() {
  int state = digitalRead(BTN_PIN);

  // HIGH -> LOW is the moment of pressing
  if (lastState == HIGH && state == LOW) {
    Serial.println("press event");
  }

  lastState = state;
  delay(20);      // crude debounce, see the proper version elsewhere
}
```

That `delay(20)` is papering over contact bounce. Do it properly, see [[Button debouncing]].

ESP32 pin modes available

|Mode|Effect|
|---|---|
|`INPUT`|No resistor, floating unless you add one externally|
|`INPUT_PULLUP`|~45kΩ to 3V3 inside the chip|
|`INPUT_PULLDOWN`|~45kΩ to GND inside the chip|
|`OUTPUT`|Drives the pin|

`INPUT_PULLDOWN` exists on the ESP32 and does work, unlike on an Arduino Uno where it does not exist at all. Still prefer pull-up.

The pins that will catch you out

|Pin|Problem|
|---|---|
|GPIO34, 35, 36, 37, 38, 39|**Input only, and no internal pull-up or pull-down whatsoever**|
|GPIO0|Held LOW at boot puts the chip into flash mode|
|GPIO2|Must not be held HIGH at boot on some boards|
|GPIO12|HIGH at boot sets the wrong flash voltage, board may not start|
|GPIO15|LOW at boot suppresses the boot log|
|GPIO6-11|Connected to the internal SPI flash, unusable|

GPIO34-39 is the big one. `pinMode(34, INPUT_PULLUP)` compiles without error and does absolutely nothing, because those pins physically lack the resistor. The pin floats and reads noise, and the code looks perfect. If you must use them for a button, fit a real 10kΩ resistor externally.

Safe pins for buttons: 4, 5, 13, 14, 16, 17, 18, 19, 21, 22, 23, 25, 26, 27, 32, 33.

Using an interrupt instead of polling

Polling in `loop()` misses presses when something else is busy. An interrupt fires the moment the pin changes.

```cpp
#define BTN_PIN 4

volatile bool pressed = false;
volatile unsigned long lastIsr = 0;

void IRAM_ATTR onPress() {
  unsigned long now = millis();
  if (now - lastIsr > 200) {      // ignore bounce
    pressed = true;
    lastIsr = now;
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(BTN_PIN, INPUT_PULLUP);
  attachInterrupt(BTN_PIN, onPress, FALLING);   // HIGH to LOW
}

void loop() {
  if (pressed) {
    pressed = false;
    Serial.println("press event");
  }
  // heavy work here still won't miss a press
}
```

`IRAM_ATTR` puts the handler in fast RAM, which the ESP32 requires. `volatile` tells the compiler the variable changes outside normal flow so it must not be optimised into a register. Keep the handler tiny, set a flag and get out.

The mistake everyone makes

Wiring a button between 3V3 and a pin with no resistor at all and no `INPUT_PULLUP`, then debugging the code. The pin floats when released and reads whatever the room's electrical noise suggests. Symptom: works if you hold the wire, random triggers otherwise.

Second: using GPIO34-39 with `INPUT_PULLUP` and assuming it took effect. It compiles, it does nothing.

Third: forgetting that with a pull-up the logic is inverted, so `if (digitalRead(pin))` is true when the button is *not* pressed.

Fourth: using two legs from the same side of a 4-pin tactile switch, so it reads as permanently pressed. See [[Buttons and Switches]].
