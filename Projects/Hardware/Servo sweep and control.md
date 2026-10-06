A hobby servo swinging back and forth, then driven to an exact angle by a potentiometer. Your first thing that moves.

What you'll learn

- Servos take a position command, not a speed — and they hold it
- The 50Hz pulse-width protocol behind every hobby servo
- Why motors need their own power supply
- Why grounds must be joined between separate supplies

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|Breadboard + wires|1|£7|Yes|
|SG90 micro servo|1|£3|No|
|10kΩ potentiometer|1|£0.60|From the dimmer project|
|4×AA battery holder or 5V supply|1|£3|No|
|470µF electrolytic capacitor|1|£0.20|No|

Wiring

```
  EXTERNAL 5V ────────────────┬──────────────┐
  (battery pack)              │              │
                           [470µF]        ┌──┴───────┐
                              │           │  SERVO   │
                              │           │ red      │
  GND rail ───────────────────┴───────────┤ brown    │
      │                                   │ orange   │
      │                                   └────┬─────┘
      │        ESP32                           │
      │     ┌──────────┐                       │
      └─────┤  GND     │                       │
            │  GPIO13  ├───────────────────────┘  signal
            │  GPIO34  ├───── pot wiper
            │  3V3     ├───── pot outer 1
            │  GND     ├───── pot outer 2
            └──────────┘
```

|From|To|Note|
|---|---|---|
|Servo brown/black wire|GND rail|servo ground|
|Servo red wire|External 5V+|**not** the ESP32's 5V pin|
|Servo orange/yellow wire|ESP32 GPIO13|the signal line, 3.3V is enough|
|External 5V−|GND rail|**this joint is mandatory**|
|ESP32 GND|GND rail|ties both supplies to one reference|
|470µF cap +|External 5V+|absorbs the current spike when it moves|
|470µF cap −|GND rail|electrolytics are polarised, get this right|
|Pot outer 1 / wiper / outer 2|3V3 / GPIO34 / GND|angle knob|

Two things that will get you:

**The ESP32 cannot power a servo.** An SG90 draws 100-200mA moving and can spike to 700mA when it stalls. The USB regulator on a dev board won't supply that; the voltage sags, the board browns out, and it reboots mid-sweep. Looks like a software bug, isn't one. Give the servo its own 5V.

**Grounds must be joined.** If the servo's supply and the ESP32 don't share a ground, "3.3V" on the signal wire is measured against a different zero and the servo sees noise or nothing at all. One wire between the two grounds fixes it. The positives stay separate; the negatives always join.

The capacitor across the servo supply is cheap insurance — it's a local reservoir that smooths the current spike when the motor kicks. Electrolytic caps are polarised; the stripe is negative, and backwards they vent.

How it works

A hobby servo is a small DC motor, a gearbox, a potentiometer measuring the output shaft, and a control circuit — all in one box. You don't tell it how fast to spin. You tell it what angle you want, and its internal loop drives the motor until the pot reads that angle, then holds it there, actively fighting anything that pushes it. That's a closed loop, the same idea you'll meet properly in [[Feedback and PID control]].

The command is a pulse repeated every 20ms (50Hz):

|Pulse width|Angle|
|---|---|
|500µs|0°|
|1500µs|90° (centre)|
|2500µs|180°|

Those numbers are nominal. Cheap SG90s vary — yours may only reach 170°, or buzz at the extremes because it's straining against its own end stop. If it buzzes, back the range off.

The gearbox trades speed for torque, which is [[Torque and Gearing]]. An SG90 gives about 1.8 kg·cm, so it'll swing a small arm or a sensor mount, not much more. More in [[Servo Motors]], and the underlying pulse generation is [[PWM output (ESP32)]].

Code

```cpp
/*
1. ESP32Servo handles the 50Hz pulse timing for us
2. attach() takes the pin plus the min/max microsecond range
3. sweep mode steps the angle a degree at a time, non-blocking
4. pot mode maps the knob straight onto 0-180
5. the servo needs time to travel, so we step rather than jump
*/

#include <ESP32Servo.h>

const int SERVO_PIN = 13;
const int POT_PIN   = 34;      // ADC1

Servo servo;

bool sweepMode = true;         // flip to false to drive it from the pot

int  angle = 0;
int  step  = 1;
unsigned long lastMove = 0;
const unsigned long MOVE_EVERY = 15;   // ms per degree, lower = faster sweep

void setup() {
  Serial.begin(115200);

  // ESP32 has 16 PWM channels; the library needs timers allocated
  ESP32PWM::allocateTimer(0);

  servo.setPeriodHertz(50);            // standard hobby servo frame rate
  servo.attach(SERVO_PIN, 500, 2500);  // pin, min pulse us, max pulse us

  servo.write(90);                     // start centred
  delay(500);                          // give it time to actually get there
}

void loop() {
  if (millis() - lastMove < MOVE_EVERY) return;
  lastMove = millis();

  if (sweepMode) {
    angle += step;

    if (angle >= 180) { angle = 180; step = -1; }   // bounce off the ends
    if (angle <= 0)   { angle = 0;   step =  1; }

    servo.write(angle);

  } else {
    int raw = analogRead(POT_PIN);            // 0..4095
    int target = map(raw, 0, 4095, 0, 180);

    // ignore tiny wobbles so ADC noise doesn't make the servo jitter
    if (abs(target - angle) > 2) {
      angle = target;
      servo.write(angle);
      Serial.printf("angle %d\n", angle);
    }
  }
}
```

Install **ESP32Servo** from the Library Manager — the plain `Servo` library is AVR-only and won't compile here.

Make it better

- Two servos on a pan-tilt bracket, one pot each, for a camera or sensor mount
- Add the HC-SR04 from [[Ultrasonic distance sensor]] and sweep it to plot a radar map
- Ease the motion with a smoothstep curve instead of linear steps — much nicer to watch
- Control it from the browser using the dashboard from [[WiFi sensor dashboard]]
- Try a continuous-rotation servo: same signal, but 1500µs now means *stop* rather than centre
