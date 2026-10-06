A decoupling capacitor is a tiny local reservoir of charge sitting right next to a chip, so when the chip suddenly gulps current it drinks from the reservoir instead of dragging the whole supply rail down.

Why the rail sags at all

Your power supply is not connected to the chip by a perfect wire. Every centimetre of jumper wire and breadboard contact has a small resistance and, more importantly, a small inductance. Inductance resists sudden changes in current.

So when the ESP32's WiFi radio switches on and demands an extra 300mA in a microsecond, the wire physically cannot deliver it that fast. The voltage at the chip dips. If it dips below about 2.8V, the brownout detector fires and the chip resets.

```
   Without a capacitor:

   3.3V ─────┐         ┌─────────
             │         │
   2.8V ─ ─ ─│─ ─ ─ ─ ─│─ ─ ─ ─ ─  brownout threshold
             └─────────┘
             ^ WiFi transmits, rail collapses, board resets


   With a capacitor:

   3.3V ─────┐     ┌───────────
             └─────┘
             ^ small dip, chip carries on
```

The capacitor's job is to supply that burst locally, from a few millimetres away, while the slow supply catches up.

Why it must be close

This is the part that gets ignored and it is the whole point.

The problem you are solving is the inductance of the wire between the supply and the chip. If you put the capacitor 10cm away down a jumper wire, the current from the capacitor has to travel through 10cm of wire, which has exactly the same inductance problem you were trying to avoid.

|Distance from the chip's power pins|Effectiveness|
|---|---|
|Under 5mm|Excellent|
|1-2cm|Good, fine on a breadboard|
|5cm|Marginal|
|10cm+|Nearly useless|

Get it within a couple of centimetres. On a breadboard that means in the columns immediately next to the module's VCC and GND pins, not over on the power rail at the far end.

The two-capacitor rule

You use two capacitors together because they cover different speeds.

|Capacitor|Handles|Why|
|---|---|---|
|100nF ceramic|Fast, high frequency spikes|Small, low internal resistance, reacts in nanoseconds|
|100µF - 470µF electrolytic|Slow, bulk sag|Holds a lot of charge, but reacts too slowly for fast spikes|

A big electrolytic alone will not stop high frequency noise, because its internal resistance and its own inductance make it sluggish. A small ceramic alone does not hold enough charge for a sustained draw. Use both.

```
   3V3 ──────┬────────┬──────────── to chip VCC
             │        │
            ─┴─      ─┴─  +
            ─┬─      ─┬─  470µF
        100nF│        │   -
             │        │
   GND ──────┴────────┴──────────── to chip GND
             ^        ^
          right at    near where
          the chip    power enters
```

|From|To|Note|
|---|---|---|
|3V3 rail|100nF ceramic leg 1|no polarity|
|100nF ceramic leg 2|GND rail|within 2cm of the chip's VCC pin|
|3V3 rail|470µF electrolytic + leg (long)|near the power input|
|470µF electrolytic - leg (short, striped)|GND rail|**polarity matters, backwards = it pops**|

Capacitors go **across** the supply, in parallel. Never in series with a power wire. If you put one in line, current stops flowing once it charges and nothing downstream works.

What value for what

|Situation|Capacitors|
|---|---|
|Any IC's power pins|100nF ceramic, always|
|ESP32 module VCC|100nF + 100µF|
|Where power enters the board|470µF electrolytic|
|Motor supply rail|470µF - 1000µF electrolytic|
|Servo supply|470µF electrolytic|
|Across a brushed motor's terminals|100nF ceramic, soldered right at the motor|
|Voltage regulator input|10µF|
|Voltage regulator output|10µF, or check the datasheet|
|Stepper driver VMOT|100µF, **mandatory, the driver dies without it**|

The 100nF-on-every-chip rule is not a suggestion, it is standard practice on every commercial board in existence. Look at any PCB and you will see a small capacitor next to every IC.

The ESP32 brownout, in practice

`Brownout detector was triggered` in the serial monitor is the single most common ESP32 message and it is almost always a power problem, not a code problem.

|Cause|Fix|
|---|---|
|Cheap USB cable, thin wires|Try a different, shorter cable. This is genuinely often it|
|Laptop USB port too weak|Use a proper 2A phone charger|
|WiFi transmit spike|100µF across 3V3 and GND on the board|
|Motor or servo on the same supply|Separate supply, plus bulk capacitance|
|Long jumper wires to power|Shorten them|
|Powering a motor from the `3V3` pin|Do not do this, see [[Voltage Regulators]]|

Try the cable first. A surprising proportion of "my ESP32 keeps resetting" is a charge-only or thin USB cable that cannot deliver half an amp without dropping half a volt.

Motor noise, a separate problem

Brushed motors are electrically filthy. The brushes physically spark as they slide across the commutator, throwing broadband noise onto the supply and radiating it into nearby wires.

Symptoms: sensor readings go wild when the motor runs, buttons trigger themselves, I2C devices stop responding, the ESP32 resets or behaves erratically.

Three fixes, use all of them:

```
   Motor with full noise suppression:

              ┌────┤├────┐         100nF across the terminals
              │          │
   6V ────────┴──[ M ]───┴──────
                   │
              ┌────◀|────┐         flyback diode elsewhere in the loop
```

|Fix|Where|Why|
|---|---|---|
|100nF ceramic across the motor terminals|Soldered at the motor itself|Kills brush sparking at the source|
|470µF - 1000µF across the motor supply|At the driver|Stops the rail sagging on current spikes|
|Flyback diode|Across the motor|Kills the switch-off spike, see [[Flyback diode protection]]|
|Separate motor supply from logic|Two supplies, one shared ground|Keeps the mess away from the ESP32|
|Twist the two motor wires together|Along their length|Cancels the radiated field|

The 100nF at the motor has to be at the motor, soldered to its terminals. Putting it on the breadboard means the noisy wire between the motor and the board is still radiating.

Grounding

|Rule|Why|
|---|---|
|One common ground for everything|Signals are voltages measured relative to ground|
|Keep motor ground return separate until it meets the supply|Motor current through a shared wire shifts the logic ground|
|Short, thick ground wires|Resistance in the ground path creates voltage offsets|
|Never rely on a breadboard rail for high current|~1A per contact|

The subtle one is the second row. If motor current returns through the same jumper wire your sensor uses as ground, the resistance of that wire means the sensor's ground is now at a slightly different voltage to the ESP32's ground while the motor runs. Your sensor readings shift. Route motor ground back to the supply on its own wire.

Testing whether you have a power problem

|Test|How|Meaning|
|---|---|---|
|Watch the rail|Multimeter on 3V3 and GND while the load runs|A visible dip means you need more capacitance|
|Swap the cable|Different, shorter USB cable|Fixes a surprising number of cases|
|Comment out the load|Disable the motor in code, run everything else|If the problem vanishes, it is power|
|Separate supplies|Power the load from a bench supply or battery|Confirms the diagnosis|
|Add a capacitor|470µF across the rail|If the symptom improves, you have your answer|

Disabling the brownout detector is not a fix. There is a way to turn it off in software and you will find it suggested online. It stops the message, and then the chip runs at a voltage where its behaviour is undefined and it corrupts data instead of resetting cleanly. Fix the power.

The mistake everyone makes

Putting the decoupling capacitor "somewhere on the power rail" instead of right next to the chip. The whole purpose is to shorten the path the burst current takes. Ten centimetres away, it does very little.

Second: assuming a big electrolytic covers everything. It is too slow for high frequency noise. Pair it with a 100nF ceramic.

Third: fitting the electrolytic backwards. The stripe and the short leg mark the negative side. Reversed, it heats, gasses and pops.

Fourth: debugging code for hours when `Brownout detector was triggered` is right there in the serial output telling you it is a power problem.
