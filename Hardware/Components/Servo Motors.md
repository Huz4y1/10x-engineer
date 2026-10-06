A hobby servo is a DC motor, a gearbox, a position sensor and a control circuit in one box, you tell it an angle and it goes there and holds.

That is the useful difference from a plain DC motor. You do not control speed, you control **position**, and the servo does the work of getting there and staying there even if something pushes back.

The three wires

|Colour (common)|Alternative|Goes to|
|---|---|---|
|Brown|Black|GND|
|Red|Red|5V supply, **not the ESP32 3V3**|
|Orange|Yellow or white|ESP32 GPIO, signal|

Signal draws essentially no current, it is just a timing pulse. All the actual power comes down the red wire from your supply.

How it is controlled

A servo listens for a pulse roughly every 20ms (50Hz). The **width** of that pulse sets the angle. The gap between pulses does nothing.

|Pulse width|Angle|
|---|---|
|1.0ms|0°|
|1.5ms|90°, centre|
|2.0ms|180°|

```
   1.0ms                                    0 degrees
   ┌──┐                                  ┌──┐
   │  │                                  │  │
 ──┘  └──────────────────────────────────┘  └──
   |<---------------- 20ms ---------------->|


   1.5ms                                    90 degrees
   ┌───┐                                 ┌───┐
   │   │                                 │   │
 ──┘   └─────────────────────────────────┘   └──


   2.0ms                                    180 degrees
   ┌────┐                                ┌────┐
   │    │                                │    │
 ──┘    └────────────────────────────────┘    └──
```

The 1.0 to 2.0ms figures are nominal. Real cheap servos often want 0.5ms to 2.5ms to hit their true end stops, and they vary between units. You calibrate by trial.

Important for 3.3V: the servo's signal input is a logic input on a 5V chip, and almost all of them read a 3.3V pulse as HIGH perfectly well. So you can drive a 5V servo's signal pin directly from an ESP32 GPIO. This is one of the cases where 3.3V into a 5V device is fine. The reverse, a 5V output into an ESP32 pin, is not. See [[Level shifting 3V3 and 5V]].

Wiring it

```
   5V (own supply) ──┬──────────── SERVO red
                     │
                   ─┴─ 470µF
                   ─┬─
                     │
   GND ──────────────┼──────────── SERVO brown
                     │
   ESP32 GND ────────┘

   ESP32 GPIO13 ───────────────── SERVO orange
```

|From|To|Note|
|---|---|---|
|5V supply +|Servo red wire|separate supply or buck converter output|
|5V supply -|Servo brown wire|-|
|5V supply -|ESP32 GND|**shared ground, without this it will not work**|
|ESP32 GPIO13|Servo orange wire|3.3V signal is fine|
|Across 5V and GND|470µF electrolytic|+ leg to 5V, absorbs the current spikes|

Do not power a servo from the ESP32's `3V3` pin. Do not power it from the `5V` pin either if it is doing real work, since that is USB power going through a thin trace, and a USB port only gives 500mA.

Current draw, the reality

|State|Current (SG90 micro servo)|Current (MG996R metal gear)|
|---|---|---|
|Idle, no load|~10mA|~50mA|
|Moving, light load|100 - 250mA|500mA - 1A|
|Holding against force|200 - 400mA|1A+|
|Stall|~700mA|**2.5A**|

An SG90 is just about survivable from a decent USB supply. An MG996R absolutely is not. And note that servos draw a spike at the moment they start moving, which is what causes the ESP32 to brown out and reset.

Code

The ESP32Servo library handles the timing for you. Install "ESP32Servo" by Kevin Harrington in the library manager, the plain "Servo" library is AVR-only and will not compile.

```cpp
#include <ESP32Servo.h>

Servo myServo;

void setup() {
  myServo.setPeriodHertz(50);              // standard 50Hz servo
  myServo.attach(13, 500, 2400);           // pin, min µs, max µs
}

void loop() {
  myServo.write(0);      // full one way
  delay(1000);
  myServo.write(90);     // centre
  delay(1000);
  myServo.write(180);    // full the other way
  delay(1000);
}
```

The `500, 2400` are the pulse widths in microseconds. Tune these per servo. If it buzzes at the extremes it is straining against its own end stop, so narrow the range until the buzzing stops.

Moving smoothly rather than snapping:

```cpp
#include <ESP32Servo.h>

Servo myServo;

void sweepTo(int target, int currentPos, int stepDelay) {
  int step = (target > currentPos) ? 1 : -1;
  for (int a = currentPos; a != target; a += step) {
    myServo.write(a);
    delay(stepDelay);
  }
}

void setup() {
  myServo.setPeriodHertz(50);
  myServo.attach(13, 500, 2400);
  myServo.write(90);
}

void loop() {
  sweepTo(180, 90, 15);
  delay(500);
  sweepTo(90, 180, 15);
  delay(500);
}
```

Types of servo

|Type|Range|What the signal means|Use for|
|---|---|---|---|
|Standard (positional)|0 - 180°|Absolute angle|Arms, steering, pan/tilt|
|Continuous rotation|Infinite|Speed and direction. 1.5ms = stop|Wheels on a small robot|
|Digital servo|0 - 180°|Same, but faster and stronger|When you need holding torque|
|High torque metal gear|0 - 180°|Same|Load bearing, MG996R etc|

A continuous rotation servo is a positional servo with the feedback pot disconnected and the gear stop removed. `write(90)` means stop, above 90 is one direction, below 90 is the other, and it never reaches a target. Handy because it is a geared motor with a built-in driver for the price of a servo.

Picking one

|Situation|Servo|
|---|---|
|Learning, light plastic parts|SG90, ~£2, 1.8 kg-cm|
|Slightly more force, still small|MG90S, metal gears|
|Robot arm joint, real load|MG996R, ~10 kg-cm, needs a real supply|
|Wheels|Continuous rotation, or use [[DC Motors]] with an H-bridge|

Torque is quoted in kg-cm, which means kilograms of force at one centimetre from the shaft. 1.8 kg-cm means it can lift 1.8kg at 1cm, or 180g at 10cm. Lever arms are brutal, work it out before you build the arm.

The mistake everyone makes

Powering the servo from the ESP32's `3V3` or `5V` pin. It twitches, the board resets, the serial output shows a boot log. Every single time this is a power problem, not a code problem. Separate supply, common ground, 470µF capacitor.

Second: forgetting the common ground when you do add a separate supply. Servo does nothing, or jitters randomly.

Third: forcing a servo past its mechanical limits in code. `write(200)` on a 180° servo makes it grind against its own stop, draw stall current, get hot and strip its gears. Clamp your values.

Fourth: assuming a servo holds position for free. Holding against a load draws real current continuously, which is why a "stationary" robot arm can flatten a battery.
