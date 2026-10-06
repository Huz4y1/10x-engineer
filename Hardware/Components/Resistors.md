A resistor is a deliberate bottleneck for current, it turns the extra electrical pressure you don't want into a tiny bit of heat.

Think of voltage as pressure and current as flow. A resistor is a narrow pipe. Ohm's law is the whole story:

```
V = I × R        I = V / R        R = V / I
```

Volts, amps, ohms. If you have 3.3V across a 220Ω resistor, current is 3.3 / 220 = 0.015A = 15mA.

Schematic symbol

```
  ───[220Ω]───
```

Older schematics use a zigzag instead of a box. Same part.

Reading the colour bands

Hold the resistor with the gold or silver band on the right, then read left to right. Most hobby resistors have 4 bands: digit, digit, multiplier, tolerance.

|Colour|Digit|Multiplier|
|---|---|---|
|Black|0|×1|
|Brown|1|×10|
|Red|2|×100|
|Orange|3|×1k|
|Yellow|4|×10k|
|Green|5|×100k|
|Blue|6|×1M|
|Violet|7|×10M|
|Grey|8|-|
|White|9|-|
|Gold|-|÷10, 5% tolerance|
|Silver|-|÷100, 10% tolerance|

Worked example

Red, Red, Brown, Gold = 2, 2, ×10 = 220Ω at 5% tolerance. That is the LED resistor you will use a thousand times.

Brown, Black, Orange, Gold = 1, 0, ×1k = 10,000Ω = 10kΩ. That is the pull-up resistor.

Honestly, buy a cheap multimeter and just measure them. Band colours under bad lighting are a nightmare and orange/red/brown all look identical on a blue-bodied resistor.

E12 values, the ones that actually exist

Resistors are not made in every value. They come in a standard series so that the 10% tolerance bands overlap and cover everything. E12 is the common one:

|Decade|Values|
|---|---|
|Ones|1.0, 1.2, 1.5, 1.8, 2.2, 2.7, 3.3, 3.9, 4.7, 5.6, 6.8, 8.2|

Multiply that row by 1, 10, 100, 1k, 10k, 100k. So 22Ω, 220Ω, 2.2kΩ, 22kΩ all exist, 250Ω does not.

The ones worth having in a drawer:

|Value|What you use it for|
|---|---|
|100Ω|Bright LED, or series protection on a signal line|
|220Ω|Default LED resistor on 3.3V and 5V|
|330Ω|Slightly dimmer LED, very common alternative|
|1kΩ|Base resistor for an NPN transistor|
|4.7kΩ|I2C pull-ups, DS18B20 data line|
|10kΩ|Pull-up / pull-down on buttons, the default|
|100kΩ|High side of a voltage divider, low current sensing|

Picking one

Two things matter: the value and the power rating.

Power is P = V × I, or P = I² × R. A 220Ω resistor with 15mA through it burns 0.015² × 220 = 0.05W. Standard hobby resistors are 1/4W (0.25W), so you have 5x headroom. Fine.

Where it bites: if you put a resistor directly across a supply. 3.3V across 10Ω is 330mA and 1.1W, which will cook a 1/4W resistor and probably brown the breadboard.

Resistors have no polarity. They go in either way round.

Series and parallel

|Arrangement|Formula|Example|
|---|---|---|
|Series (end to end)|R1 + R2|220 + 220 = 440Ω|
|Parallel (side by side)|(R1 × R2) / (R1 + R2)|220 ∥ 220 = 110Ω|

Useful when you only own one value. Two 220Ω in series gets you 440Ω, close enough to 470Ω for an LED.

The mistake everyone makes

Assuming the resistor value has to be exact. It almost never does. For an LED, anything from 100Ω to 1kΩ on 3.3V will work, it just changes brightness. For a pull-up, anything from 4.7kΩ to 47kΩ works. Stop hunting for the perfect value, grab the nearest one and move on.

The second mistake is putting the resistor in the wrong place and thinking it matters. In a simple series loop, current is the same everywhere, so the resistor limits the LED whether it sits before or after it. Both are correct.
