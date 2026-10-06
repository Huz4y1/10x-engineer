Anything with a coil in it fights back when you switch it off, generating a voltage spike many times the supply voltage, and a diode wired backwards across the coil gives that spike somewhere harmless to go.

Why the spike happens

A coil stores energy in a magnetic field. Current through the coil builds that field, and the field holds the energy.

The rule that matters: **current through an inductor cannot change instantly.** If you suddenly open the switch, the coil still has current flowing and nowhere for it to go, so it generates whatever voltage it takes to keep that current moving. With an open circuit in the way, "whatever it takes" can be hundreds of volts.

```
   V = L × (dI/dt)
```

If the current goes from 0.5A to zero in a microsecond, `dI/dt` is 500,000 amps per second. Multiply by even a small inductance and the voltage is enormous.

That spike appears across the transistor that just switched off. Your MOSFET is rated for maybe 30V, and the spike is 200V. It punches through. Sometimes it dies instantly, sometimes it degrades slightly on every switch until it fails weeks later, which is the more infuriating outcome.

What has a coil in it

|Component|Inductive?|Needs a flyback diode?|
|---|---|---|
|Relay coil|Yes|**Yes**|
|DC motor|Yes|**Yes**|
|Solenoid / valve|Yes|**Yes**|
|Stepper motor|Yes|Built into the driver|
|Transformer|Yes|Yes|
|LED strip|No|No|
|Heater element|No|No|
|Buzzer (passive piezo)|No|No|
|Buzzer (active, has a coil)|Sometimes|Yes, if it clicks|

If it moves something magnetically, it has a coil.

Where the diode goes and which way round

Directly across the coil terminals, **reversed** compared to normal current flow. Cathode (the striped end) to the positive side.

```
              ┌────◀|────┐         stripe points UP, at +12V
              │  1N4007  │
   12V ───────┴──[COIL]──┴─────┐
                               │ D
   GPIO16 ──[220Ω]─────────────┤ G    IRLB8721
                               │ S
   GND ────────────────────────┴────── ESP32 GND
```

|From|To|Note|
|---|---|---|
|12V supply +|Coil terminal 1|-|
|12V supply +|Diode cathode (striped end)|same node as coil terminal 1|
|Coil terminal 2|MOSFET drain|-|
|Coil terminal 2|Diode anode (no stripe)|same node as the drain|
|ESP32 GPIO16|220Ω resistor|-|
|220Ω resistor|MOSFET gate|-|
|MOSFET source|GND|-|
|12V supply -|ESP32 GND|shared ground|

The rule in one line: **stripe towards positive.**

It looks wrong on purpose. Wired that way, the diode is reverse biased during normal operation and does absolutely nothing, no current, invisible. It only wakes up when the coil tries to drive current backwards.

Why it works

Trace the two states.

Transistor on. Current flows 12V, through the coil, through the transistor, to ground. The diode has 12V on its cathode and roughly 0V on its anode, so it is reverse biased and blocks. It is not in the circuit at all.

Transistor off. The coil insists on maintaining its current. It flips its own voltage polarity trying to keep the current going, and the end connected to the drain shoots up above 12V. Once it gets 0.7V above the supply, the diode conducts and the coil's current circulates round the loop of coil and diode. That current decays over a few milliseconds as the resistance burns off the stored energy.

```
   Transistor ON               Transistor OFF

   12V ──┬──────┐              12V ──┬──────┐
         │      │                    │      │
        ◀|      │                   ◀|◄─┐   │    diode now conducting
   blocked      │                    │  │   │
         └[COIL]┘                    └[COIL]┘    current loops harmlessly
            │                           │
          ── ON ──                   -- OFF --
            │                           │
           GND                         GND
```

The spike is clamped to about 12.7V, supply plus one diode drop. Your 30V MOSFET never notices.

Choosing the diode

|Requirement|Rule|
|---|---|
|Current rating|At least the load's normal running current|
|Voltage rating|At least the supply voltage, ideally 2×|
|Speed|Fast enough to catch the spike|

|Part|Rating|Speed|Good for|
|---|---|---|---|
|1N4001|1A, 50V|Slow|Relays, low speed switching|
|1N4007|1A, 1000V|Slow|Relays, general purpose, cheap|
|1N4148|200mA, 100V|Fast|Small signal relays only|
|1N5819|1A, 40V|Fast Schottky|**Motors, PWM, anything switching quickly**|
|SS34|3A, 40V|Fast Schottky|Bigger motors|

For a relay switched a few times a second, a 1N4007 is fine and costs nothing. For a motor under PWM at 1kHz, use a Schottky like the 1N5819. A slow diode does not turn on quickly enough to catch each spike, so part of it gets through, and at 1000 switches a second that adds up.

The classic circuits

Relay driven by an NPN:

```
   5V ────────┬──────────┐
              │          │
             ◀|─ 1N4007  │
              │          │
              └─[COIL]───┘
                   │
                   │ C
   GPIO16 ─[1kΩ]───┤ B      2N2222
                   │ E
   GND ────────────┴──────── ESP32 GND
```

|From|To|Note|
|---|---|---|
|5V supply +|Relay coil pin 1 and diode cathode|stripe here|
|Relay coil pin 2|Transistor collector and diode anode|-|
|ESP32 GPIO16|1kΩ resistor|-|
|1kΩ resistor|Transistor base|-|
|Transistor emitter|GND|-|
|5V supply -|ESP32 GND|shared ground|

DC motor driven by a MOSFET:

```
   6V ────────┬──────────┐
              │          │
             ◀|─ 1N5819  │      Schottky, fast
              │          │
              └──[ M ]───┘
                   │
                   │ D
   GPIO16 ─[220Ω]──┤ G      IRLB8721
                   │ S
                   ├─[10kΩ]─ GND
                   │
   GND ────────────┴──────── ESP32 GND
```

|From|To|Note|
|---|---|---|
|6V supply +|Motor terminal 1 and diode cathode|stripe to positive|
|Motor terminal 2|MOSFET drain and diode anode|-|
|ESP32 GPIO16|220Ω to MOSFET gate|-|
|MOSFET gate|10kΩ to GND|stops it spinning at boot|
|MOSFET source|GND|-|
|6V supply -|ESP32 GND|shared ground|

Also add a 100nF ceramic directly across the motor terminals. That handles the brush sparking noise, which is a separate problem to the flyback spike. See [[Decoupling capacitors and clean power]].

The reversible motor case

A single flyback diode only works if current always flows one way. In an H-bridge the motor is driven in both directions, so a single diode would short the supply in one of them. You need four, one across each transistor.

Good news: every H-bridge module has them built in. DRV8833, TB6612, L298N all include internal protection diodes. You do not add anything. See [[H-Bridge motor driver]].

What happens without one

|Load|What breaks|How fast|
|---|---|---|
|Small relay|Transistor degrades|Weeks or months|
|DC motor|MOSFET punches through|Days, or immediately|
|Solenoid|Transistor dies|Often on the first switch off|
|Any of them|ESP32 resets, glitches, corrupt readings|Immediately|

That last row is the one people never connect to the cause. The spike does not just hit the transistor, it radiates onto every nearby wire and back down the supply rails. Symptoms are an ESP32 that reboots when a relay clicks, sensor readings that go wild, and serial output filling with junk. It looks like a software bug. It is a missing diode.

Checking it is the right way round

Multimeter on diode test mode. Red probe on the coil terminal that goes to the positive supply, black probe on the terminal that goes to the transistor. You should read **OL** or open circuit, meaning the diode is reverse biased in normal operation, which is correct.

Swap the probes and you should get a reading around 0.5V to 0.7V.

If you get 0.6V the first way round, it is backwards. Turn it round now, before you power up, because a forward-fitted flyback diode is a direct short across the supply and something will burn within seconds.

The mistake everyone makes

Fitting it "the sensible way", matching normal current flow. It short circuits the supply through the coil, the diode glows, and either it or the transistor is destroyed almost immediately. The flyback diode is meant to look wrong. Stripe towards positive.

Second: leaving it out entirely because the circuit "works fine". It does work fine. The damage is cumulative and invisible until the transistor fails for no apparent reason.

Third: using a slow 1N4007 on a PWM'd motor. It cannot switch fast enough to catch every spike at 1kHz. Use a Schottky.

Fourth: putting the diode across the transistor instead of across the coil. It has to be across the coil, the thing that generates the spike.
