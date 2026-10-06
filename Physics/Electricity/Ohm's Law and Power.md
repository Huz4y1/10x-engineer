Two formulas cover about 90% of the electrical maths you'll ever do on a hobby build.

The two that matter

V = I × R  (volts = amps × ohms)

P = V × I  (watts = volts × amps)

That's it. Everything else is rearranging them.

|If you know|And you want|Use|
|---|---|---|
|V and R|Current|I = V / R|
|V and I|Resistance|R = V / I|
|I and R|Voltage|V = I × R|
|V and I|Power|P = V × I|
|I and R|Power|P = I² × R|

Worked example, plain resistor

A 220Ω resistor sat across 5V.

I = V / R = 5 / 220 = 0.0227A = 22.7mA

P = V × I = 5 × 0.0227 = 0.11W

A standard quarter-watt (0.25W) resistor handles 0.11W fine. Warm, not dangerous.

Worked example, sizing an LED resistor

This is the single most common calculation in hobby electronics.

A red LED has a forward voltage of about 2.0V, meaning it always drops 2.0V across itself no matter what, and you want to run it at 10mA off a 3V3 pin.

Voltage left for the resistor = 3.3 − 2.0 = 1.3V

R = V / I = 1.3 / 0.01 = 130Ω

Nothing sells 130Ω, so grab the next one up, 150Ω or the classic 220Ω. Going higher just means a slightly dimmer LED, which is always safe.

```
  3V3 ───[150Ω]───▶|─── GND
                   LED
```

Check the resistor won't cook: P = 1.3 × 0.01 = 0.013W. Nothing.

Forward voltages differ by colour, so keep this handy:

|LED colour|Forward voltage|Resistor from 3V3 at 10mA|Resistor from 5V at 10mA|
|---|---|---|---|
|Red|~2.0V|130Ω → use 150Ω|300Ω → use 330Ω|
|Yellow|~2.1V|120Ω → use 150Ω|290Ω → use 330Ω|
|Green|~2.8V|50Ω → use 68Ω|220Ω|
|Blue / white|~3.2V|too tight, run from 5V|180Ω → use 220Ω|

Never wire an LED straight across a supply with no resistor. An LED barely resists current at all, so V = I × R with a tiny R means a huge I, and it dies instantly or takes your GPIO pin with it.

Why things get hot

Power is voltage across a part multiplied by current through it, and that power comes out as heat. So heat happens wherever you have *both* a voltage drop and a decent current.

Worked example, a linear regulator

You drop 12V down to 5V with an AMS1117 to feed a board pulling 500mA.

Voltage dropped by the regulator = 12 − 5 = 7V

P = 7 × 0.5 = 3.5W

3.5W in a part the size of a grain of rice. It will hit well over 100°C and shut down or die. This is exactly why you use a switching buck converter instead of a linear regulator when the input voltage is much higher than the output.

Worked example, why thin wire melts

A motor pulling 5A through a wire with 0.2Ω of resistance:

P = I² × R = 5² × 0.2 = 25 × 0.2 = 5W

5W spread along a thin wire is enough to melt the insulation. Note the I², doubling the current quadruples the heat, which is why motor wiring is always thicker than signal wiring.

Why you care when wiring a robot

Every LED, every current-limiting resistor, every "why is this component hot" question comes back to these two formulas. If a part is hot, work out the voltage across it and the current through it, multiply, and you'll know whether it's normal or about to fail.
