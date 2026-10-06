A diode is a one-way valve for current, it conducts in one direction and blocks in the other.

That is genuinely the whole concept. Current can go through the triangle and out of the bar. Try to push it the other way and the diode says no.

Schematic symbol

```
  anode ───▶|─── cathode
                 (band on the body)
```

On a real diode there is a painted stripe at one end. **The stripe is the cathode**, and it matches the bar in the symbol. Remember that and you will never fit one backwards.

Forward voltage drop

A diode is not a perfect wire when conducting. It eats a fixed amount of voltage:

|Type|Forward drop|Speed|Typical part|Use|
|---|---|---|---|---|
|Silicon signal|0.7V|Fast|1N4148|Logic, small signals, up to 200mA|
|Silicon rectifier|0.7V|Slow|1N4001 - 1N4007|Mains rectifying, flyback on relays|
|Schottky|0.2V - 0.4V|Very fast|1N5819, SS34|Reverse protection, buck converters, fast flyback|
|Zener|Reverse breakdown at a set voltage|-|1N4733 (5.1V)|Voltage clamping / reference|
|LED|1.8V - 3.4V|-|-|Light, see [[LEDs]]|

The 0.7V drop is not optional and it is not tunable. If you put a 1N4001 in series with a 5V supply to protect against reverse polarity, your circuit now sees 4.3V. That matters if you are feeding a 5V servo. A Schottky only costs you 0.3V, which is why they get used for exactly that job.

If you own three diodes, own these: **1N4148** (signal), **1N4007** (general purpose power), **1N5819** (Schottky).

Reading the markings

Diodes have the part number printed on the glass or black body, plus the stripe. `1N4148` is a small orange glass bead. `1N400x` is a fat black barrel. A Schottky often looks like the black barrel but grey.

Current rating is in the part number's family, not the marking. 1N4148 is 200mA, 1N4001 is 1A, 1N5819 is 1A. Pick one rated for more current than you will push through.

What you actually use them for

Flyback / freewheeling protection

The big one. Any coil (relay, motor, solenoid) generates a large reverse voltage spike when you cut the current. A diode across the coil, wired **backwards** to normal current flow, gives that spike a harmless loop to burn itself out in.

```
              ┌────◀|────┐        diode reversed across the coil
              │  1N4007  │
   5V ────────┴──[COIL]──┴────── to transistor collector
                                  then GND
```

|From|To|Note|
|---|---|---|
|5V rail|Coil pin 1|also diode cathode (striped end)|
|Coil pin 2|Diode anode|and to the transistor/driver|
|Diode stripe|Points at 5V|backwards for normal current, this is correct|

Full explanation in [[Flyback diode protection]].

Reverse polarity protection

Put a diode in series with your battery input. Wire the battery backwards by accident and the diode blocks it, saving everything downstream.

```
  BATT+ ───▶|─── VIN of board
            1N5819

  BATT- ───────── GND
```

|From|To|Note|
|---|---|---|
|Battery +|Diode anode (no stripe)|current flows this way|
|Diode cathode (stripe)|Board VIN|circuit sees Vbatt - 0.3V|
|Battery -|Board GND|-|

Costs you 0.3V with a Schottky. Cheap insurance if you are swapping battery packs often.

Isolating two power sources

Two diodes, one from USB 5V and one from a battery, both feeding the same rail. Whichever is higher wins, and neither can backfeed into the other. Crude but effective.

```
   USB 5V ───▶|───┬─── to circuit
                  │
   BATT   ───▶|───┘
```

|From|To|Note|
|---|---|---|
|USB 5V|Diode 1 anode|-|
|Battery +|Diode 2 anode|-|
|Both cathodes|Circuit VIN|higher source supplies, lower is blocked|

Zener diodes, briefly

A Zener is used **backwards** on purpose. Below its rated voltage it blocks like a normal diode. Above it, it conducts and holds the voltage there. Useful for clamping an input so it can never exceed 3.3V.

```
   input ───[1kΩ]───┬─── ESP32 GPIO
                    │
                   ◀|─  3.3V Zener (stripe up)
                    │
                   GND
```

|From|To|Note|
|---|---|---|
|Input signal|Resistor leg 1|resistor absorbs the excess|
|Resistor leg 2|ESP32 GPIO and Zener cathode|striped end goes to the signal|
|Zener anode|GND|-|

This is a safety net, not a design. Use a proper [[Voltage dividers]] arrangement for known signals.

The mistake everyone makes

Fitting the flyback diode the "sensible" way round, matching normal current flow. It will conduct straight away, short the coil supply out, and either cook the diode or the transistor within seconds. The flyback diode looks wrong. It is meant to look wrong. Stripe towards the positive rail.

Second: forgetting the 0.7V drop exists, then wondering why the "5V" line downstream of your protection diode measures 4.3V and the sensor is misbehaving.
