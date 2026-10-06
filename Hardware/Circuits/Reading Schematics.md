A schematic is a map of what connects to what, it deliberately says nothing about where anything physically sits.

That is the mental shift. A schematic is not a picture of your breadboard. Two components drawn next to each other might be at opposite ends of the board, and a wire drawn as a long line might be a 5mm jumper. Only the connections are real.

The symbol vocabulary

|Symbol|Component|Notes|
|---|---|---|
|`───[220Ω]───`|Resistor|Value always shown. No polarity|
|`────┤├────`|Capacitor, non-polarised|Ceramic, either way round|
|`────┤(────`|Capacitor, polarised|Curved plate is negative|
|`───▶\|───`|Diode or LED|Current flows anode to cathode, bar is the block|
|`───o  o───`|Switch, open|Momentary or latching|
|`───[BTN]───`|Button|Shorthand used throughout these notes|
|`───[COIL]───` or `─(M)─`|Inductor / motor|Anything with a winding|
|`───[10kΩ]───`|Pull-up or pull-down|Value tells you which job|
|`3V3`, `5V`, `VCC`|Positive supply|Always drawn at the top|
|`GND`, `⏚`|Ground|Always drawn at the bottom|
|`──┬──`|Junction, connected|The dot or T is the connection|
|`──│──` crossing|Crossing, **not** connected|No dot means no join|

The junction rule is the one that trips people up. Two lines crossing on a schematic are only connected if there is a dot or a T shape. A plain `+` crossing means the wires pass over each other.

```
   ────┬────       connected

   ────│────       NOT connected
       │
```

Conventions that make schematics readable

|Convention|Why|
|---|---|
|Positive supply at the top|Current is drawn as flowing downhill|
|Ground at the bottom|Everything returns here|
|Signals flow left to right|Input on the left, output on the right|
|Ground symbols repeated everywhere|All ground symbols on a page are the same node|

That last one matters enormously. A schematic does not draw one long wire back to the battery. It just puts a `GND` label wherever a connection goes to ground. **Every GND label on the schematic is the same physical point.** Same for `3V3`. This is how a complicated schematic stays readable, and it is why a beginner sees "twelve grounds" and does not realise they are one node.

How to trace a circuit

Pick a point and follow it. Two techniques.

Trace by node. Put your finger on a wire and follow it in every direction until you hit a component. Everything you touched on the way is one electrical node, one point, one voltage. Then list what components attach to that node. Repeat for the next node. A circuit is just a list of nodes and what hangs off each.

Trace the current loop. Start at the positive supply, follow the path forward through each component, and see where it eventually reaches ground. If there is no complete path from + to -, nothing happens. If there is a path with nothing in it, that is a short and something burns.

Worked example

```
                    3V3
                     │
                  [10kΩ]
                     │
                     ├───────── ESP32 GPIO4
                     │
                   [BTN]
                     │
                    GND
```

Trace it by node:

|Node|What connects to it|
|---|---|
|3V3|Top of the 10kΩ|
|Middle node|Bottom of the 10kΩ, GPIO4, top of the button|
|GND|Bottom of the button|

Three nodes. Now reason about it. Button open, no current flows through the resistor, so no voltage is dropped across it, so the middle node sits at 3V3 and the pin reads HIGH. Button pressed, the middle node is tied directly to GND so it reads LOW, and the 10kΩ limits the current from 3V3 to ground to 0.33mA which is nothing.

That is the entire analysis and it took two sentences. Full version in [[Button with Pull-up and Pull-down]].

Schematic to breadboard

The translation is mechanical once you think in nodes.

|Schematic concept|Breadboard equivalent|
|---|---|
|A node (a wire with things on it)|One column of 5 holes|
|Two components joined by a wire|Both legs in the same column|
|A `GND` label|A jumper to the `-` rail|
|A `3V3` label|A jumper to the `+` rail|
|A crossing with no dot|Two different columns, no shared hole|

So the schematic above becomes:

|From|To|Note|
|---|---|---|
|ESP32 3V3|`+` rail|-|
|ESP32 GND|`-` rail|do this first|
|`+` rail|Resistor leg 1 (hole A5)|-|
|Resistor leg 2 (hole B10)|-|now column 10 is the middle node|
|Column 10|Button leg 1|shares the node|
|Column 10|Jumper to ESP32 GPIO4|shares the node|
|Button leg 2|Jumper to `-` rail|opposite pair of the switch|

Three things landed in column 10, which is exactly the three-connection middle node from the schematic. That mapping is the whole skill.

Things schematics do not tell you

|Not shown|You have to know|
|---|---|
|Physical size or package|A resistor symbol could be any size part|
|Which leg is which on a transistor|Look up the pinout, symbols do not match leg order|
|Wire lengths or routing|Up to you|
|Decoupling capacitors, sometimes|Often omitted as understood, add them anyway|
|Power connections on ICs|Often hidden to reduce clutter|

That last one is a genuine surprise. On a real schematic, an op-amp or logic chip is drawn as a triangle or box with only its signal pins. The VCC and GND pins are drawn separately in a corner, or not at all. They still have to be wired.

Datasheet pinouts

The schematic symbol for a transistor tells you base, collector, emitter. It does not tell you which physical leg is which, and there is no universal standard. A 2N2222 and a BC547 have their legs in different orders despite being near-identical parts.

Always check the datasheet pinout, and note whether the drawing is viewed from the **front** (flat face towards you) or the **back**. Getting that wrong mirrors the whole thing.

Reading a module's silkscreen

Most of your parts are breakout modules, not bare components. The printed labels are your schematic.

|Label|Means|
|---|---|
|`VCC`, `V+`, `+`|Power in. **Check 3.3V or 5V**|
|`GND`, `G`, `-`|Ground|
|`SDA` / `SCL`|I2C data and clock|
|`MOSI` `MISO` `SCK` `CS`|SPI|
|`TX` / `RX`|Serial, cross them over: module TX to ESP32 RX|
|`OUT`, `DO`, `S`, `SIG`|Digital output from the module|
|`AO`|Analogue output, use an ADC1 pin|
|`EN`, `CE`|Enable|
|`INT`|Interrupt, fires when something happens|

TX to RX is the one people get wrong. Serial connections cross over. Module TX goes to ESP32 RX, module RX goes to ESP32 TX. I2C and SPI do not cross, SDA goes to SDA.

The mistake everyone makes

Trying to lay out the breadboard to physically look like the schematic drawing. The schematic is a topological map, not a floor plan. Build by node, not by picture.

Second: seeing many `GND` symbols on a schematic and thinking they are separate grounds. They are all one point.

Third: assuming a crossing means a connection. No dot, no join.

Fourth: wiring a transistor from the schematic symbol without checking the real leg order in the datasheet.
