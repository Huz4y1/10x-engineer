An ADC turns a voltage into a number, and the two things that decide whether that number is any good are how finely it can divide the voltage and how often it looks.

What an ADC actually does

ADC stands for analog-to-digital converter, and it's built into every hobby microcontroller. You point it at a pin, it measures the voltage, and it hands you an integer.

Picture a ruler. The ADC lays its ruler against the voltage and reports which mark it's nearest. How many marks the ruler has is the resolution, and that's set by the number of bits.

number of steps = 2^bits

|Bits|Steps|Step size on a 3.3V range|
|---|---|---|
|8|256|12.9 mV|
|10 (Arduino Uno)|1024|3.2 mV|
|12 (ESP32, Pico)|4096|0.81 mV|
|16 (ADS1115 external)|65536|0.05 mV|
|24 (HX711 load cell amp)|16.7 million|0.0000002 V|

Worked example, 12-bit on 3.3V

A 12-bit ADC gives 4096 steps, numbered 0 to 4095.

step size = 3.3 / 4095 = 0.000806V = 0.806 mV

So the smallest voltage change it can possibly notice is about 0.8mV. Anything finer is invisible to it.

Converting a reading back to volts:

voltage = reading × 3.3 / 4095

A reading of 2048 gives 2048 × 3.3 / 4095 = 1.65V, which is exactly half of 3.3V as you'd expect.

```cpp
const float VREF  = 3.3f;
const int   STEPS = 4095;          // 12-bit, 0..4095

float readVolts(int pin)
{
    int raw = analogRead(pin);     // 0 .. 4095
    return raw * VREF / STEPS;
}
```

Turning that into something useful, say a temperature sensor putting out 10mV per °C:

```cpp
float readTempC(int pin)
{
    float v = readVolts(pin);
    return v / 0.010f;             // 10 mV per degree
}
```

Reference voltage matters more than bits

The ADC measures against a reference voltage, usually the chip's own 3V3 supply. If that supply sags to 3.2V when the motors kick in, every reading you take is now wrong by 3%, and no amount of extra bits helps. A stable reference beats a high bit count almost every time. This is one more reason for decoupling caps, see [[Capacitance and RC timing]].

Also worth knowing: the ESP32's built-in ADC is noticeably non-linear near the top and bottom of its range and needs calibration. If you actually care about accuracy, an external ADS1115 over I2C is a couple of pounds and far better.

Match the resolution to the range you use

If your sensor only ever swings between 1.0V and 1.5V, you're using 0.5V of a 3.3V range, so you're only really getting about 620 of your 4096 steps. Amplify the signal to fill the range, or reduce the ADC's reference, and you get the resolution back.

Sample rate and aliasing

Resolution is how finely you measure. Sample rate is how often, in samples per second (Hz).

Sample too slowly and you don't just get a rougher version of the signal, you get a *wrong* one. That's aliasing.

The everyday version is a wagon wheel in a film appearing to spin backwards, or slowly, or stand still. The camera takes 24 pictures a second, the wheel spins much faster than that, and the brain reconstructs a completely fictional slow rotation from the snapshots. Nothing about the picture tells you it's wrong.

The rule is that you must sample at more than twice the highest frequency present in the signal. That's the Nyquist rate. In practice aim for 5 to 10 times, because the theoretical minimum only just works.

|Signal|Highest frequency|Minimum sample rate|Sensible rate|
|---|---|---|---|
|Room temperature|~0.01Hz|0.02Hz|1Hz|
|Human audio|20kHz|40kHz|44.1kHz (why CDs use it)|
|Robot IMU for balancing|~50Hz|100Hz|200-500Hz|
|Motor current spikes|kHz|kHz+|As fast as you can|

The nasty part is that a fast signal you don't care about, motor PWM noise at 20kHz say, will alias down into your slow measurement band and appear as a slow drifting wobble that looks completely real. You cannot remove it in software afterwards, because by then it's indistinguishable from real data.

The fix is a low-pass filter in hardware *before* the ADC, called an anti-aliasing filter. A resistor and capacitor is often enough, see [[Capacitance and RC timing]] and [[Noise and Filtering]].

Why you care when building

Sample at a steady rate. Calling analogRead inside a loop with a bunch of other code means your interval jitters, and any filter or derivative you compute from it is then wrong.

Don't chase bits when your problem is noise or an unstable reference, which it usually is.

If a reading shows a slow drift or wobble that isn't physically there, suspect aliasing from something fast nearby.

---

**Going deeper:** [[24 — SCIENTIFIC COMPUTING]] - floating point and precision. [[PID and Kalman filters]] - why control loop rate matters.
