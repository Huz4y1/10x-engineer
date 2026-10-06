A breadboard is a grid of hidden metal clips that join certain holes together, so pushing two component legs into the same group of holes wires them together without solder.

The entire skill is knowing which holes are already joined. Everything confusing about breadboards comes from getting that wrong.

The internal connections

```
   ┌───────────────────────────────────────────────────────┐
   │  +  ●───●───●───●───●───●───●───●───●───●───●───●     │  power rail, all joined
   │  -  ●───●───●───●───●───●───●───●───●───●───●───●     │  ground rail, all joined
   │                                                       │
   │        1   2   3   4   5   6   7   8   9  10          │
   │     A  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │        │   │   │   │   │   │   │   │   │   │          │
   │     B  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │  columns joined
   │        │   │   │   │   │   │   │   │   │   │          │  vertically in
   │     C  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │  groups of 5
   │        │   │   │   │   │   │   │   │   │   │          │
   │     D  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │        │   │   │   │   │   │   │   │   │   │          │
   │     E  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │                                                       │
   │  ═════════════ CENTRE CHANNEL, nothing crosses ══════ │
   │                                                       │
   │     F  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │        │   │   │   │   │   │   │   │   │   │          │
   │     G  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │        │   │   │   │   │   │   │   │   │   │          │
   │     H  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │        │   │   │   │   │   │   │   │   │   │          │
   │     I  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │        │   │   │   │   │   │   │   │   │   │          │
   │     J  ●   ●   ●   ●   ●   ●   ●   ●   ●   ●          │
   │                                                       │
   │  +  ●───●───●───●───●───●───●───●───●───●───●───●     │
   │  -  ●───●───●───●───●───●───●───●───●───●───●───●     │
   └───────────────────────────────────────────────────────┘
```

The three rules that follow from that picture:

|Rule|Meaning|
|---|---|
|A1-B1-C1-D1-E1 are one node|Five holes in a column, all the same electrical point|
|F1-G1-H1-I1-J1 are a **different** node|The centre channel breaks the column in half|
|A1 and A2 are **not** connected|Rows do not connect sideways|
|The whole `+` rail is one node|Runs the full length of the board|
|`+` and `-` rails are separate|Obviously, but people bridge them by accident|

Why the centre channel exists

So that a DIP chip can straddle it. Push an 8-pin chip across the gap and each of its legs lands in its own 5-hole column, giving you four free holes per pin to wire into. If the channel were not there, opposite legs of the chip would be shorted together.

This is also why a 4-pin tactile switch must straddle the channel. See [[Buttons and Switches]].

The rail break, which catches everyone

On many full-size breadboards the power rails are **not** continuous end to end. There is a break in the middle, sometimes marked by a gap in the printed blue and red lines.

```
   +  ●──●──●──●──●──●   ╳   ●──●──●──●──●──●
   -  ●──●──●──●──●──●   ╳   ●──●──●──●──●──●
                       break
```

If your circuit works at one end of the board and is dead at the other, this is why. Fix it with two jumper wires bridging the gap on both rails, or just check with a multimeter on continuity before you assume.

Half-size 400-point boards usually have continuous rails. Full-size 830-point boards usually do not. Look for the printed line stopping.

The ESP32 problem

A standard ESP32 DevKit is 25.4mm wide across the pin rows. A standard breadboard has 30 columns split by a channel, giving you 5 usable holes each side... except the ESP32 is wide enough that on many breadboards it covers **all** the holes on one side, leaving you nothing to plug into.

|Situation|Fix|
|---|---|
|Board covers all holes one side|Use two breadboards side by side, ESP32 straddling both|
|Board is a narrow variant|Fine, you get one free row each side|
|Only one breadboard|Plug the ESP32 in at the very end so one row hangs off, or use jumpers from the top|

The two-breadboard trick is standard practice, not a hack. Snap the plastic side clips together, remove the power rail strip between them, and the ESP32 sits across the join with holes available on both sides.

A worked layout

Building the LED circuit from [[LED current limiting circuit]] on an actual board:

```
   ESP32 GPIO2 ────────────────► row 10 (top half)
                                    │
                        [220Ω] spans row 10 to row 15
                                    │
                                 row 15 ── LED anode
                                    │
                                 row 16 ── LED cathode
                                    │
                            jumper row 16 to the (-) rail
                                    │
                    ESP32 GND ──────► (-) rail
```

|Step|Holes used|Note|
|---|---|---|
|1|ESP32 GND to `-` rail|do this first, always|
|2|Jumper GPIO2 to hole A10|-|
|3|Resistor leg 1 in B10, leg 2 in B15|spans two separate columns|
|4|LED anode (long) in C15|shares column 15 with the resistor|
|5|LED cathode (short) in C16|-|
|6|Jumper D16 to `-` rail|closes the loop|

Notice that the connection between the resistor and the LED is made by them sharing column 15. That is the whole trick. Two legs in the same column of five are wired together.

Wire colour discipline

Not decoration, this is how you avoid destroying things.

|Colour|Use for|
|---|---|
|Red|Positive supply, 3V3 or 5V|
|Black|Ground, always and only|
|Any other|Signals|

Pick this up early. When a board is a mess of thirty wires, being able to see at a glance that black means ground stops you plugging 5V into a GPIO.

Good practice

|Do|Why|
|---|---|
|Connect grounds first|Nothing works without a common ground|
|Power off while rewiring|Prevents momentary shorts as a wire brushes past|
|Trim component legs short|Long legs flop around and touch each other|
|Keep wires flat against the board|Easier to trace, less chance of pulling one out|
|Use the rails for power, always|Do not run 3V3 from column to column with jumpers|
|Test with a multimeter on continuity|Two seconds beats an hour of debugging|

What breadboards cannot do

|Limit|Consequence|
|---|---|
|~1A per contact|Motors and ESCs must not go on a breadboard, see [[Brushless Motors and ESCs]]|
|Contacts wear out|An old board develops intermittent holes that look like code bugs|
|Stray capacitance between rows|Anything above a few MHz behaves oddly|
|Nothing holds firmly|A knocked wire is the single most common "it stopped working"|

If a circuit worked yesterday and does not today, and you have not changed the code, suspect the breadboard before anything else. Pull each wire and reseat it. Loose contacts are the number one cause of intermittent hardware faults.

The mistake everyone makes

Assuming the row of five holes runs across the board rather than along it. If your LED does nothing, check that its two legs are not sitting in the same column of five, which shorts it out, and check they are not in unrelated columns with no connection at all.

Second: forgetting the mid-board rail break, then spending an hour on a circuit whose power rail simply is not connected at that end.

Third: plugging both legs of a component into the same column. That connects the component to itself and does nothing. Components must span two different columns.

Fourth: relying on the ESP32's own pins for a ground connection to a second power supply. The `-` rail must physically join both.
