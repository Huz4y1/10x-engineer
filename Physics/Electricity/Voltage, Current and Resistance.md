Electricity makes a lot more sense if you stop thinking about electrons and picture water moving through pipes instead.

The water analogy

Picture a water tank up on a roof, a pipe coming down from it, and a narrow bit of pipe somewhere along the way.

|Electrical thing|Water version|What it actually means|
|---|---|---|
|Voltage (V, volts)|Pressure pushing the water|How hard the electricity is being shoved along|
|Current (I, amps)|How much water flows past per second|How much charge is actually moving|
|Resistance (R, ohms, Ω)|A narrow section of pipe|How much the circuit fights the flow|
|Ground (GND)|The drain everything empties into|The 0V reference everything is measured against|

Three things that follow from the analogy and confuse everyone at first:

Voltage is always *between* two points. There is no such thing as "the voltage at this wire" on its own, only the voltage between that wire and something else, usually GND. Pressure only means something as a difference.

Current is the same all the way round one loop. Water doesn't pile up inside the pipe, so whatever leaves the battery comes back to it. If 100mA goes into an LED, 100mA comes out the other side.

Resistance turns flow into heat. A narrow pipe heats up as water is forced through it. That is exactly what a resistor does, and it is why things burn.

Units you'll actually type

|Unit|Symbol|Common sizes you'll see|
|---|---|---|
|Volt|V|3V3, 5V, 12V, 3.7V (LiPo cell)|
|Amp|A|1A motor, 500mA servo|
|Milliamp|mA|1A = 1000mA, an ESP32 idles ~80mA|
|Ohm|Ω|220Ω LED resistor|
|Kilohm|kΩ|10kΩ pull-up, 1kΩ = 1000Ω|

A simple circuit

```
  3V3 ───[220Ω]───▶|─── GND
                   LED
```

3V3 is the pressure source, the 220Ω is the narrow pipe limiting flow, the LED is the thing doing work, GND is the drain. Break any part of that loop and nothing flows at all.

What you actually measure with a multimeter

|Measuring|How|Gotcha|
|---|---|---|
|Voltage|Circuit powered on, black probe on GND, red probe on the point you care about. Probes sit *across* the thing, in parallel|Forgetting the black probe needs a common ground|
|Current|Circuit powered on, but you have to break the wire and put the meter *in the loop* so all the current flows through it|Leaving the meter in amps mode and then probing a voltage shorts it out and blows the meter fuse|
|Resistance|Power OFF, and ideally the component lifted out of the circuit|Other parts in parallel give you a nonsense reading|
|Continuity|Power off, meter beeps if there's a connection|Best tool you own for finding a bad solder joint|

Why you care when wiring a robot

Your board says 3V3 on one pin and 5V on another. Those are pressure levels. Feeding a 5V sensor output into a 3V3 input pin is over-pressuring it and can kill the pin.

Your regulator says 800mA. That is a flow limit. A servo that pulls 1.5A when it stalls will brown out the whole board and reset your microcontroller mid-run.

Every ground must be tied together. If your motor battery and your logic board don't share a GND, the two circuits have no common reference and signals between them mean nothing.
