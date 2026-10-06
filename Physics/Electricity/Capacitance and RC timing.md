A capacitor is a tiny rechargeable bucket of charge sat across your power rail.

The bucket analogy

A battery is a reservoir. A capacitor is a small bucket plumbed into the pipe right next to whatever is drinking. When the load suddenly gulps, the bucket empties into it instantly while the reservoir catches up, then it refills. It holds far less energy than a battery, but it can dump and refill it thousands of times a second, which a battery cannot.

Capacitance (measured in farads, F) is just how big the bucket is.

|Unit|Size|Typical use|
|---|---|---|
|pF (picofarad)|0.000000000001F|Crystal load caps, RF|
|nF (nanofarad)|1000pF|100nF is the standard decoupling cap|
|µF (microfarad)|1000nF|1µF to 100µF, general smoothing|
|mF / big µF|470µF to 4700µF|Bulk caps next to motor drivers|

A capacitor blocks steady DC once it's full, but it happily passes sudden changes. That one sentence explains most of what caps are used for.

Charging is a curve, not a ramp

Charge a capacitor through a resistor and it fills fast at first, then slower and slower, because as it fills there's less voltage difference left to push more charge in. Same as a water tank filling from a pipe when the pressure difference shrinks.

The one formula you need is the time constant:

τ = R × C     (tau, in seconds, with R in ohms and C in farads)

|Time elapsed|Capacitor is charged to|
|---|---|
|1τ|63%|
|2τ|86%|
|3τ|95%|
|5τ|99%, call it full|

Discharging is the mirror image: after 1τ it's dropped to 37% of where it started.

Worked example

R = 10kΩ = 10,000Ω, C = 100µF = 0.0001F

τ = 10,000 × 0.0001 = 1 second

So charging from a 5V rail: after 1s it's at 3.15V, after 3s it's at 4.75V, after 5s call it 5V.

```
  5V ───[10kΩ]───┬─── to ADC pin
                 │
              [100µF]
                 │
                GND
```

Worked example, button debouncing

A mechanical button doesn't switch cleanly, the contacts bounce and your GPIO sees five or ten fake presses over a couple of milliseconds. An RC filter smooths that out.

R = 10kΩ, C = 100nF = 0.0000001F

τ = 10,000 × 0.0000001 = 0.001s = 1ms

1ms of smoothing kills most bounce. Bump the cap to 1µF for a 10ms constant if the button is really nasty. You can also do this purely in software, but hardware debounce costs two parts and zero CPU.

Why caps sit right next to chips

Every microcontroller pin that switches state pulls a sharp spike of current for a few nanoseconds. The wire from the regulator has resistance and inductance, so it can't deliver that spike fast enough, and the chip's supply voltage momentarily dips. Enough dips and the chip resets or reads garbage.

A 100nF ceramic cap soldered as close to the chip's VCC and GND pins as physically possible is a local bucket that supplies those spikes instantly. This is called a decoupling or bypass cap.

|Cap|Where it goes|Job|
|---|---|---|
|100nF ceramic|Right at every chip's power pins, as short as you can|Kills fast switching spikes|
|10µF ceramic|Near each module or regulator output|Medium-speed dips|
|470µF+ electrolytic|Across the motor driver's power input|Absorbs the huge gulps when motors start|

If you skip these, your project works on the bench and then randomly resets when the motors kick in, and you'll spend a week blaming your code.

Why you care when wiring a robot

Motors browning out the board is the number one cause of random resets, and bulk capacitance across the motor supply fixes most of it.

Electrolytic caps are polarised. The stripe is the negative leg. Put one in backwards and it will pop, sometimes loudly.

A big charged cap holds energy after you unplug things. On low-voltage hobby stuff it's harmless, but it's why a board's LED stays lit for a second after power off.
