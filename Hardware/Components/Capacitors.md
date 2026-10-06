A capacitor is a tiny rechargeable battery that charges and discharges almost instantly, it smooths out sudden changes in voltage.

The mental model that actually helps: a capacitor is a small water tank plumbed into your supply line. When something suddenly gulps a load of current, the tank drains a bit instead of the pressure in the whole house dropping. When the demand stops, it refills.

That is 90% of what you use them for on an ESP32 project. The chip draws current in violent bursts when the WiFi radio transmits, and without a local tank the supply voltage dips and the chip resets.

Schematic symbols

```
  Non-polarised (ceramic)      Polarised (electrolytic)

  ────┤├────                   ────┤(────
                                   +  -
```

The curved plate on the polarised symbol is the negative side.

Ceramic vs electrolytic

|  |Ceramic|Electrolytic|
|---|---|---|
|Looks like|Small orange/blue blob|Little metal can|
|Polarity|None, either way round|Yes, gets this wrong and it pops|
|Typical values|1pF to 1µF|1µF to 4700µF|
|Marking|3 digit code|Value printed in plain text|
|Good at|Fast, high frequency noise|Bulk energy storage|
|Leakage|Basically none|Slowly self discharges|
|Lifespan|Forever|Dries out over years, especially hot|

You will mostly use two: a 100nF ceramic and a 100µF or 470µF electrolytic. Together they cover fast noise and slow sag.

Reading the markings

Electrolytics are easy, the can says `470µF 16V` and has a stripe down the negative side. The negative leg is also the shorter one.

Ceramics use a 3 digit code that works exactly like resistor bands, in picofarads:

|Code|Meaning|Value|Also written|
|---|---|---|---|
|104|10 × 10⁴ pF|100,000pF|100nF or 0.1µF|
|103|10 × 10³ pF|10,000pF|10nF|
|102|10 × 10²pF|1,000pF|1nF|
|105|10 × 10⁵ pF|1,000,000pF|1µF|
|22|literal|22pF|22pF|

`104` is the one you will see constantly. It is the standard decoupling capacitor.

Unit ladder, because the same value gets written three ways:

|Unit|Relative|
|---|---|
|1F|farad, huge, only supercapacitors|
|1µF|one millionth of a farad|
|1nF|0.001µF|
|1pF|0.000001µF|

What value for what job

|Job|Value|Type|Where it goes|
|---|---|---|---|
|Decoupling a chip|100nF|Ceramic|As close to the chip's VCC and GND pins as physically possible|
|Bulk supply smoothing|100µF to 470µF|Electrolytic|Across the power rails near where power enters the board|
|Motor / servo brownout fix|470µF to 1000µF|Electrolytic|Across the motor supply rails|
|Hardware button debounce|100nF|Ceramic|Pin to GND, with a series resistor|
|Regulator input/output|10µF|Ceramic or electrolytic|Datasheet will specify|
|Crystal load caps|22pF|Ceramic|Either side of the crystal|

Voltage rating

Every capacitor has a maximum voltage. Pick one rated at least double your supply. On a 5V rail use a 16V part, on a 12V motor rail use a 25V or 35V part. They are the same price. Undervolting an electrolytic makes it fail, sometimes loudly.

Polarity, and why it matters

An electrolytic capacitor has a chemical layer that only forms in one direction. Wire it backwards and it heats, gasses and either quietly leaks or pops the top open with a bang and a smell you will remember. Ceramics do not care.

```
  5V ──────┬──────── to circuit
           │
          ─┴─  +
          ─┬─  470µF        stripe/short leg to GND
           │   -
  GND ─────┴────────
```

|From|To|Note|
|---|---|---|
|5V rail|Capacitor + leg (long)|no stripe on this side|
|Capacitor - leg (short)|GND rail|stripe on the can marks this leg|

Capacitors go across the supply, in parallel, never in series with it. If you put a capacitor in line with a power wire, nothing works after it charges.

The mistake everyone makes

Putting the decoupling capacitor "somewhere on the rail" instead of right next to the chip. The whole point is to shorten the path the burst current takes. A 100nF capacitor 10cm away down a jumper wire does almost nothing, because the wire's own inductance is the problem you were trying to solve. Get it within a couple of centimetres.

The other one: a charged 470µF capacitor still holds voltage after you unplug the battery. That is why the LED fades instead of snapping off. Not dangerous at 5V, but do not be surprised when your circuit runs for half a second after power off.
