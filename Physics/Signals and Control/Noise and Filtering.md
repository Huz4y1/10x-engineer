Every real sensor reading is the value you wanted plus a bit of rubbish, and filtering is how you get rid of the rubbish without destroying the signal.

Where noise comes from

|Source|What it looks like|Fix|
|---|---|---|
|Motors and PWM switching|Fast spiky rubbish that appears exactly when the motors run|Separate power rails, caps, twisted wires|
|Long unshielded wires|Wires act as antennas, picking up whatever's around|Shorten them, twist signal with its ground|
|Shared ground return|Motor current flowing through a ground trace shifts the reference under your sensor|Star ground, single joining point|
|Mains hum|A steady 50Hz (or 60Hz) wobble|Move away from mains cables, filter it out|
|Power supply ripple|Reading drifts with load|Decoupling and bulk caps|
|The ADC and sensor themselves|A jittery last digit or two|Averaging, oversampling|
|Loose or cold solder joints|Wild intermittent garbage|Reflow the joint. This one is not a filtering problem|

Fix it in hardware first. Filtering in software costs you response time, and it can never recover information that the noise already destroyed. Hardware fixes worth doing before you write a line of filter code:

Decoupling caps at every chip, see [[Capacitance and RC timing]].

Motor supply and logic supply kept separate, joined only at one ground point, see [[Batteries and Power budgets]].

100nF across the motor terminals, and a flyback diode, see [[Inductance and Back-EMF]].

An RC low-pass right at the ADC pin. A 1kΩ and a 100nF gives a cutoff around 1.6kHz, which kills PWM noise and passes anything slow.

Averaging, the simplest thing that works

Take N readings back to back and average them. Random noise partly cancels itself, and the improvement goes with the square root of N, so 4 samples halves the noise, 16 samples quarters it. Diminishing returns get you fast.

```cpp
int readAveraged(int pin, int n)
{
    long sum = 0;

    for (int i = 0; i < n; i++)
    {
        sum += analogRead(pin);
    }

    return sum / n;
}
```

Cheap and effective, but it blocks while it samples. Fine for a temperature reading, bad inside a control loop.

Moving average

Keep the last N readings in a buffer and average whatever's currently in it. You get one smoothed output per new sample rather than one per N samples, so it doesn't block.

```cpp
const int N = 8;

int   buf[N] = {0};
int   idx    = 0;
long  sum    = 0;

int movingAverage(int newSample)
{
    sum -= buf[idx];        // drop the oldest value out of the running total
    buf[idx] = newSample;   // overwrite it with the new one
    sum += newSample;

    idx = (idx + 1) % N;    // wrap around the circular buffer

    return sum / N;
}
```

Bigger N means smoother output but more lag: the average is always dragging behind by roughly half the window. With N = 8 at 100Hz sampling, you're about 40ms behind reality. In a fast control loop that lag will make things oscillate, see [[Feedback and PID control]].

Exponential low-pass, the one you'll actually use

Rather than storing a buffer, just blend each new reading a little bit into the running value. This is an exponential moving average, and it's a first-order low-pass filter written out in one line.

```cpp
float filtered = 0.0f;
const float ALPHA = 0.1f;   // 0 = ignore new data, 1 = no filtering at all

float lowPass(float newSample)
{
    filtered = (ALPHA * newSample) + ((1.0f - ALPHA) * filtered);
    return filtered;
}
```

One float of state, one multiply-add, no buffer, no loop. It's what almost every hobby project should be using.

|ALPHA|Behaviour|Use for|
|---|---|---|
|0.5|Barely filters, very responsive|Fast control loops|
|0.1|Good general smoothing|Most sensors|
|0.02|Very smooth, visibly laggy|Battery voltage, temperature|

Low-pass means it passes slow changes and blocks fast ones. That's what you want when the signal you care about is slow (a distance reading) and the noise is fast (spikes). If ever the opposite is true, a high-pass does the reverse, and it's how you strip a slow drifting offset out of a sensor.

Always call the filter at a steady rate. ALPHA is meaningless without a consistent sample interval, because how much smoothing you get depends on how often you're blending.

A median filter is the other tool worth knowing

Averaging is terrible at rejecting a single wild outlier, because the outlier drags the average with it. Ultrasonic distance sensors are famous for this, mostly good readings with the occasional nonsense 400cm. Take three readings and use the middle one and the outlier vanishes completely.

```cpp
int median3(int a, int b, int c)
{
    if ((a <= b && b <= c) || (c <= b && b <= a)) return b;
    if ((b <= a && a <= c) || (c <= a && a <= b)) return a;
    return c;
}
```

Why you care when building

If your sensor only goes mad when the motors are running, that's electrical noise and no filter will fix it properly. Fix the wiring.

Every filter buys smoothness with lag, and lag inside a feedback loop causes oscillation. Filter as little as you can get away with.

Spikes want a median filter, fuzz wants a low-pass. Using the wrong one leaves you with a smoothed-out version of the spike.

---

**Going deeper:** [[PID and Kalman filters]] - Kalman filters as principled sensor fusion.
