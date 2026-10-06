Everything your microcontroller talks to is either a smooth varying voltage or a stream of on/off states, and the pins for each behave completely differently.

The difference

Analog is continuous. The voltage can be anything in range, 1.2V, 1.203V, whatever, and it changes smoothly. A dimmer knob, a temperature sensor, a microphone.

Digital is discrete. The voltage is only allowed to be one of two things, high or low, 1 or 0. A button, a limit switch, an I2C bus.

Think of a ramp versus a staircase. Real-world physical quantities, light, heat, pressure, sound, force, are all analog. Computers only work in digital. The job of most of your electronics is converting between the two.

|Analog|Digital|
|---|---|
|Potentiometer, LDR, thermistor|Push button, limit switch, PIR|
|Analog microphone, current sensor|I2C / SPI / UART sensors|
|Any voltage in range|Only HIGH or LOW|
|Noise directly corrupts the value|Noise is ignored until it's big enough to flip a bit|
|Needs an ADC to read|Read straight off a GPIO pin|

Digital's big advantage is noise immunity. A 50mV wobble on an analog line is 50mV of error in your reading forever. A 50mV wobble on a digital line is nothing, because anything above the threshold still reads as HIGH.

What a GPIO pin can actually see

A digital input pin is not a voltmeter. It's a comparator with two thresholds, and it has no idea about anything between them.

For a 3.3V part like an ESP32 or a Pi Pico, roughly:

|Input voltage|Pin reads|
|---|---|
|0V to about 0.8V|LOW (0)|
|about 0.8V to 2.0V|Undefined. Could read either, could flicker|
|about 2.0V to 3.3V|HIGH (1)|
|Above 3.6V|Damage. The protection diodes conduct and the pin degrades or dies|

The rough rule for most chips is below 30% of the supply is LOW and above 70% is HIGH, with a no-man's-land in the middle. Exact numbers are in the datasheet under VIL and VIH.

That middle band is why slow-changing signals are a problem. A slowly rising voltage spends time in the undefined zone and the pin can toggle several times on the way through. Chips with Schmitt trigger inputs have two different thresholds, one for going up and one for going down, which fixes this. That gap between them is called hysteresis.

Floating pins

An input pin with nothing connected is "floating". It's not 0, it's not 1, it's an antenna picking up whatever electrical noise is around, and it will read randomly.

A button connects a pin to something when pressed, but leaves it floating when released, so you need a resistor to define the released state:

```
  3V3 ───[10kΩ]───┬─── GPIO pin
                  │
                 [ ] button
                  │
                 GND
```

That's a pull-up. Released, the pin sits at 3V3 through the resistor and reads HIGH. Pressed, the button pulls it hard to GND and it reads LOW, so the logic is inverted, pressed = 0. Most microcontrollers have internal pull-ups you can switch on in software (`INPUT_PULLUP`), which saves the part.

A pull-down is the mirror image, resistor to GND, button to 3V3, pressed = 1.

Buttons also bounce, giving several fake transitions per press, see [[Capacitance and RC timing]].

Mixing 5V and 3V3

This kills more boards than anything except reversed polarity. A 5V sensor output into a 3V3 input pin is 5V on a pin rated for 3.6V.

|Direction|Safe?|Fix|
|---|---|---|
|3V3 output → 5V input|Usually fine, 3.3V is above most 5V parts' HIGH threshold|Nothing needed, but check the datasheet|
|5V output → 3V3 input|NOT safe|Voltage divider, or a proper level shifter for anything fast|

A quick divider using [[Series and Parallel circuits]]: a 10kΩ and a 20kΩ in series across the 5V signal, tapping between them, gives 5 × 20/30 = 3.33V. Fine for slow signals like a button or UART. For I2C or SPI use a real bidirectional level shifter, because a resistor divider slows the edges down too much.

Output pins have limits too

A GPIO output can only supply a small current, typically 20-40mA on an ESP32 or AVR, with a much lower total across all pins at once. That's enough for an LED with a resistor and nothing else. Motors, relays and LED strips all need a transistor or driver in between, see [[Inductance and Back-EMF]].

Why you care when building

Every input pin needs a defined state. Floating pins produce ghost button presses that look like software bugs.

Check the voltage of every sensor before you wire it in. 3V3 and 5V modules look identical.

If a digital reading is flickery, it's either floating, bouncing, or sitting in the undefined band.

PWM is a digital trick that fakes an analog output, switching a pin on and off fast so the average voltage lands where you want it. It's how you dim LEDs and set motor speed from a pin that can only do 0 and 1.
