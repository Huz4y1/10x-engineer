A multimeter is the first tool to buy and it answers the question "is the electricity actually where I think it is".

The dial

|Symbol|Mode|Measures|
|---|---|---|
|V with a straight line|DC volts|Batteries, supplies, logic pins. This is what you want 95% of the time|
|V with a wave|AC volts|Mains. Leave it alone unless you know what you're doing|
|Ω|Resistance|Resistor values, whether something is shorted|
|Speaker icon|Continuity|Beeps if two points are connected. The most-used mode|
|A|Current|Draw of a circuit. Requires rewiring, see below|
|Diode symbol|Diode test|Which way round an LED or diode goes|

Measuring voltage, in parallel

```
  Probes go ACROSS the thing, circuit stays intact.

    black probe ──> GND
    red probe   ──> the point you're curious about

         ┌──────────────┐
    3V3 ─┤              │
         │   circuit    │
    GND ─┤              │
         └──┬────────┬──┘
            │        │
          (red)   (black)
            │        │
          ┌─┴────────┴─┐
          │  DC volts  │
          └────────────┘
```

What you should see

|Point|Expected|
|---|---|
|ESP32 3V3 pin to GND|3.25 - 3.35 V|
|ESP32 VIN to GND (on USB)|4.7 - 5.1 V|
|A GPIO set HIGH|~3.3 V|
|A GPIO set LOW|~0 V|
|A floating input pin|Anything, and it wanders. That's the point|
|`INPUT_PULLUP` pin, unpressed|~3.3 V|
|AA battery|1.5 V fresh, 1.2 V tired|
|18650 lithium cell|4.2 V full, 3.7 V nominal, 3.0 V empty|

Continuity, the one you'll use most

```
  Beeps when resistance is under ~50 ohms, i.e. it's a wire.

  Use it to check:
    - is this breadboard rail actually connected end to end
      (many boards split the rails in the middle, unmarked)
    - did that solder joint take
    - is this jumper wire broken inside the insulation
    - which pin on the connector is which
    - are two tracks shorted together that shouldn't be

  ALWAYS with the power off. Continuity mode pushes a small
  current out of the probe, and on a live circuit the readings
  are meaningless.
```

Resistance

```
  Measure a resistor OUT of the circuit. In circuit, everything
  else in parallel drags the reading down and you get nonsense.

  Reading 1 or OL      -> open circuit, nothing connected
  Reading 0.0          -> a short
  Reading 220          -> a 220 ohm resistor, as expected
  Reading 10.2k on a
  marked 10k           -> fine, 5% tolerance
```

Current, in series, and this is where people break things

```
  Voltage is measured ACROSS. Current is measured THROUGH.
  The meter has to become part of the circuit.

  before                          after

  supply ──────── circuit         supply ──┬── circuit
         └──────── GND                     │      │
                                       ┌───┴──┐   │
                                       │ mA A │   │
                                       └───┬──┘   │
                                           └──────┘
                                    break the wire, meter fills the gap

  And move the RED PROBE to the A or mA socket.
  Forgetting this is the number one multimeter mistake.
```

```
  Why it matters:

  In volts mode the meter is a near-infinite resistance,
  it barely disturbs anything.

  In amps mode the meter is a near-zero resistance, a wire.

  Leave the probe in the A socket, then measure a voltage,
  and you have connected a wire straight across your supply.
  Best case the meter's fuse blows. Worst case the supply does.

  Habit: measure current, then IMMEDIATELY move the red probe back.
```

Typical current draws

|Thing|Current|
|---|---|
|ESP32 idle, WiFi off|~40 mA|
|ESP32 with WiFi transmitting|150-260 mA burst|
|ESP32 deep sleep (bare module)|~10 uA|
|ESP32 deep sleep (DevKit board)|5-20 mA, the USB chip and LED|
|An LED with a 220R resistor|~10 mA|
|A small hobby servo, moving|100-500 mA|
|A DC motor, stalled|1 A and up|

When a circuit is dead, in order

```
  1. Is there power at all?
     3V3 pin to GND. If it's 0V, nothing else matters.

  2. Is the ground connected?
     Continuity from the ESP32 GND to the component's GND.
     Missing ground is the most common single fault, and it
     produces the strangest symptoms.

  3. Is the supply sagging?
     Measure 3V3 WHILE the thing runs. 3.3V idle dropping to
     2.8V under load means the supply can't cope. That causes
     brownout reboots.

  4. Does the pin do what the code says?
     digitalWrite HIGH, measure it. 3.3V means the software
     is fine and the fault is downstream.

  5. Is anything shorted?
     Power off, continuity between 3V3 and GND. A beep is a
     short and you need to find it before powering up again.

  6. Are the components the right way round?
     Diode mode on LEDs and diodes. Electrolytic capacitors
     have a stripe on the negative side and they do explode.
```

Diode mode

```
  Red probe on the anode (long leg), black on the cathode:
    an LED lights faintly, and the display shows ~1.8-3.0

  Probes the other way: OL, nothing.

  That's how you find which leg is which on an LED you've
  already trimmed the legs off.
```

The gotcha

```
  A cheap meter measures DC volts, continuity and resistance
  perfectly well. That covers nearly everything in this section.

  What it CANNOT do:
    - see a signal changing faster than about twice a second
    - show you a PWM waveform (it shows the average, which is
      genuinely useful, but not the shape)
    - decode I2C or SPI

  For any of that you need [[Logic analyser and Oscilloscope]].

  Never put a multimeter across mains in resistance, continuity
  or current mode. Volts only, and only if you know what you're doing.
```

---

**Where it's used:** [[20 — EMBEDDED]] - debugging a circuit that won't behave.
