The hello world of hardware. One LED, one resistor, one pin turning on and off.

What you'll learn

- How a GPIO pin actually drives current
- Why an LED always needs a resistor next to it
- The `setup()` / `loop()` shape of every Arduino sketch
- Blocking delays vs doing it properly later

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|Breadboard|1|£4|Yes|
|Jumper wires|2|£3 for a pack|Yes|
|5mm LED|1|£0.10|Yes|
|220Ω resistor|1|£0.05|Yes|

Wiring

```
  ESP32
  ┌──────────┐
  │          │
  │   GPIO2  ├───[220Ω]───▶|─── GND
  │          │              LED
  │   GND    ├──────────────────┘
  └──────────┘
```

|From|To|Note|
|---|---|---|
|ESP32 GPIO2|Resistor leg 1|any output-capable pin|
|Resistor leg 2|LED anode (long leg)|220Ω limits current|
|LED cathode (short leg)|ESP32 GND|completes the circuit|

Resistors have no polarity so either leg works. The LED does — long leg to the resistor, short leg (and the flat spot on the plastic rim) to GND. Backwards it just sits there doing nothing, it won't break.

How it works

A GPIO pin set to OUTPUT is a tiny switch inside the chip. HIGH connects it to 3.3V, LOW connects it to 0V. Current flows from 3.3V, through the resistor, through the LED, into GND.

The LED is not a resistor. It doesn't obey [[Ohm's Law and Power]] — it drops a roughly fixed voltage (about 2V for a red one) and then takes as much current as you let it. Let it take too much and it cooks itself, and possibly the ESP32 pin with it. The resistor is what sets the limit.

Doing the sum: 3.3V supply − 2V across the LED = 1.3V left over for the resistor. 1.3V / 220Ω = about 6mA. Comfortably bright, comfortably under the ~12mA an ESP32 pin likes to source. See [[LEDs]] and [[Resistors]].

Never wire an LED straight from a pin to GND with no resistor. It might survive a few seconds. It might not.

Code

```cpp
/*
1. pick a pin and tell the chip it's an output
2. setup() runs once when the board powers up
3. loop() runs forever after that
4. digitalWrite drives the pin HIGH (3.3V) or LOW (0V)
5. delay() stops everything for N milliseconds
*/

const int LED_PIN = 2;   // GPIO2, most boards also have an onboard LED here

void setup() {
  pinMode(LED_PIN, OUTPUT);      // this pin will drive, not listen
  Serial.begin(115200);          // so we can print to the serial monitor
  Serial.println("blink starting");
}

void loop() {
  digitalWrite(LED_PIN, HIGH);   // pin -> 3.3V, current flows, LED on
  delay(500);                    // 500ms of nothing

  digitalWrite(LED_PIN, LOW);    // pin -> 0V, no current, LED off
  delay(500);
}
```

If nothing happens: check the LED isn't backwards, check GND is actually connected, check you selected the right board in the IDE. Roughly 90% of first-time hardware bugs are one of those three.

Make it better

- Swap `delay()` for a `millis()` based timer so the CPU isn't frozen — see [[Non-blocking timing]]
- Fade it in and out instead of hard on/off using [[PWM output (ESP32)]]
- Add a second and third LED on other pins and chase them in sequence
- Try 100Ω and 1kΩ resistors and watch the brightness change, then work out the current for each
