This is the one that quietly destroys beginner projects, so it's worth ten minutes.

Coils hate change

An inductor is just a coil of wire, and a motor winding, a relay coil, a solenoid and a buzzer are all inductors whether you think of them that way or not.

Water analogy: imagine a long, heavy pipe full of moving water. Once the water is flowing it has momentum. Slam a valve shut and the water doesn't politely stop, it hammers into the valve with enormous pressure. Plumbers call this water hammer and it bursts pipes.

A coil does exactly the same thing with current. Current flowing in a coil stores energy in a magnetic field. Cut the current off suddenly and that energy has to go somewhere, so the coil generates whatever voltage it takes to keep the current flowing, in the opposite direction to what you'd expect. That kickback voltage is called back-EMF, or the inductive spike.

Capacitors resist a change in voltage. Inductors resist a change in current. They're mirror images.

How big is the spike

V = L × (ΔI / Δt)

In plain words: the spike is the inductance multiplied by how much the current changed divided by how quickly it changed. Switch off fast enough and the number gets absurd.

Worked example

A small relay coil, L = 10mH = 0.01H, carrying 100mA, switched off by a transistor in 1 microsecond.

ΔI = 0.1A, Δt = 0.000001s

V = 0.01 × (0.1 / 0.000001) = 0.01 × 100,000 = 1000V

A thousand volts, off a 5V supply. In reality something breaks down before it gets that high, and the something is usually your transistor, which is rated for maybe 40V. It doesn't always die immediately, it degrades over hundreds of switching cycles and then fails weeks later, which makes it a horrible bug to track down.

The fix is one diode

Put a diode across the coil, backwards. In normal operation it does nothing because it's reverse-biased and blocks. The instant the coil kicks, the spike is the other polarity, so the diode conducts and gives the current a harmless loop to die away in. This is called a flyback or freewheeling diode.

```
  5V ────────┬───────────┬──────
             │           │
         [ MOTOR ]      ─┴─  striped end (cathode) points UP to 5V
             │          ─┬─  1N5819 flyback diode
             ├───────────┘
             │
        [ transistor ]
             │
  GND ───────┴──────────────────
```

The diode sits across the motor pointing UP, towards the positive rail: cathode (the striped end) to 5V, anode to the transistor side. Normally it blocks and does nothing. When the coil kicks, it conducts and the spike circulates harmlessly. Fit it the other way round and it's a dead short across your supply, and it will smoke immediately.

|Load|Diode to use|Why|
|---|---|---|
|Relay, solenoid, buzzer|1N4148 (small) or 1N4007|Slow switching, cheap parts are fine|
|Brushed DC motor|1N5819 Schottky or similar|Faster recovery, handles higher current|
|Motor driver IC (L298N, DRV8833, TB6612)|Already built in|Check the datasheet, most modern ones include them|

For brushed motors also solder a 100nF ceramic cap across the two motor terminals, and ideally from each terminal to the motor case. Brushed motors spark continuously as the brushes make and break contact, and that sparking is a constant noise generator that upsets nearby sensors and I2C buses.

Why you care when wiring a robot

Any time you switch an inductive load with a transistor, MOSFET or GPIO-driven anything, you need a flyback diode. No exceptions.

Never drive a motor or relay directly from a GPIO pin. Aside from the current, the spike goes straight back into the microcontroller.

If your ESP32 randomly resets, your I2C sensor drops out, or your transistor works for a day and then doesn't, suspect back-EMF before you suspect your code.

The same effect is useful too: boost converters deliberately switch a coil on and off to *generate* a higher voltage than they're fed. Same physics, controlled on purpose.
