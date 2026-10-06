There are only two ways to connect two components, one after the other or side by side, and they behave in opposite ways.

Series means one single path

```
  5V ───[100Ω]───[100Ω]─── GND
```

Everything has to flow through both parts, like water through two narrow bits of the same pipe.

Parallel means two separate paths

```
        ┌──[100Ω]──┐
  5V ───┤          ├─── GND
        └──[100Ω]──┘
```

The flow splits, goes down two pipes, and joins back up.

The rules

|In series|In parallel|
|---|---|
|Current is the SAME through everything|Voltage is the SAME across everything|
|Voltage SPLITS between the parts|Current SPLITS between the branches|
|Resistances add: R = R1 + R2|Resistance goes DOWN: 1/R = 1/R1 + 1/R2|
|One part fails open, everything stops|One branch fails open, the others carry on|

Worked example, series

Two 100Ω resistors in series on 5V.

Total R = 100 + 100 = 200Ω

I = V / R = 5 / 200 = 0.025A = 25mA

That same 25mA goes through both. Voltage across each = I × R = 0.025 × 100 = 2.5V. Two equal resistors split 5V evenly into 2.5V and 2.5V. That's a voltage divider, and it's how you drop a 5V sensor signal down to something a 3V3 pin can survive.

Worked example, parallel

The same two 100Ω resistors, but in parallel on 5V.

Each branch sees the full 5V, so each pulls 5 / 100 = 50mA.

Total current = 50 + 50 = 100mA, four times the series case.

Total resistance = 5 / 0.1 = 50Ω, which is half of 100Ω. Two equal resistors in parallel always give you half the value.

Why it matters for LEDs

Series LEDs share one resistor and all get the exact same current, which is what you want, but you have to have enough supply voltage to cover all their forward voltages added up. Three red LEDs at 2.0V each need 6V before you even add a resistor, so that won't run off 3V3 or 5V.

```
  12V ───[330Ω]───▶|───▶|───▶|─── GND
```

Parallel LEDs on ONE shared resistor is the classic beginner mistake:

```
                  ┌───▶|───┐
  5V ───[330Ω]────┤        ├─── GND     don't do this
                  └───▶|───┘
```

No two LEDs have identical forward voltage. The one with the slightly lower forward voltage hogs most of the current, glows brighter, gets hotter, its forward voltage drops further, and it hogs even more. Give every parallel LED its own resistor:

```
       ┌──[220Ω]───▶|──┐
  5V ──┤               ├── GND
       └──[220Ω]───▶|──┘
```

Why it matters for batteries

|Wiring|Effect|Example|
|---|---|---|
|Cells in series|Voltage adds, capacity stays the same|3 LiPo cells at 3.7V = 11.1V (a "3S" pack), still 2200mAh|
|Cells in parallel|Capacity adds, voltage stays the same|2 × 2200mAh at 3.7V = 4400mAh at 3.7V|

Series gets you the voltage a motor needs. Parallel gets you the runtime and lets the pack deliver more current without sagging. Big packs do both, a "3S2P" pack is three in series, doubled up in parallel.

Never parallel batteries that aren't at the same voltage and same chemistry. The higher one dumps current into the lower one with almost nothing limiting it, which with lithium cells means fire.

Why you care when wiring a robot

Power rails are parallel. Every module you hang off 5V is another parallel branch, and every branch adds to the total current your regulator has to supply. Add up all the branches before you assume the regulator can cope.

Anything wired in series with your load, including thin wires, long jumper leads and cheap switches, drops voltage that your load then doesn't get. That's why a motor that runs fine on the bench gets weak once it's wired through a breadboard.
