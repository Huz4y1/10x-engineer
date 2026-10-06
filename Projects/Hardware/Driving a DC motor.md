A DC motor spinning both directions at variable speed, through an L298N H-bridge. The building block of every wheeled robot after this.

What you'll learn

- Why a GPIO pin can never drive a motor directly
- How an H-bridge reverses a motor with four switches
- Speed control by PWM instead of by voltage
- Back-EMF, and what a flyback diode is protecting

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|Breadboard + wires|1|£7|Yes|
|L298N motor driver module|1|£4|No|
|TT gear motor (yellow, 3-6V)|1|£2|No|
|4×AA battery holder|1|£3|No|
|10kΩ potentiometer|1|£0.60|Yes by now|

The L298N is old, inefficient and drops about 2V across itself, but it's cheap, indestructible and has screw terminals. A TB6612FNG or DRV8833 is genuinely better if you're buying fresh. The wiring idea is identical.

Wiring

```
  BATTERY 6V ────────── L298N +12V
                          │
  ┌───────────────────────┴────────────┐
  │             L298N                  │
  │  +12V  GND  +5V                    │
  │  ENA  IN1  IN2   IN3  IN4  ENB     │
  │  OUT1 OUT2       OUT3 OUT4         │
  └───┬────┬───────────────────────────┘
      │    │
      │    │      ┌─────────┐
      └────┴──────┤  MOTOR  │
                  └─────────┘

        ESP32
     ┌──────────┐
     │  GPIO14  ├────── ENA   (PWM, speed)
     │  GPIO27  ├────── IN1   (direction)
     │  GPIO26  ├────── IN2   (direction)
     │  GPIO34  ├────── pot wiper
     │  3V3     ├────── pot outer 1
     │  GND     ├──┬─── pot outer 2
     └──────────┘  │
                   └─── L298N GND  (shared ground, mandatory)
```

|From|To|Note|
|---|---|---|
|Battery +|L298N +12V terminal|works from ~6V up|
|Battery −|L298N GND terminal|motor supply ground|
|L298N GND|ESP32 GND|**mandatory** — one common reference|
|L298N OUT1 / OUT2|Motor terminals|swap them to flip which way is "forward"|
|ESP32 GPIO14|L298N ENA|PWM here sets speed|
|ESP32 GPIO27|L298N IN1|direction bit A|
|ESP32 GPIO26|L298N IN2|direction bit B|
|Pot outer 1 / wiper / outer 2|3V3 / GPIO34 / GND|speed knob|

Leave the ENA jumper **off** — with it on, the enable pin is tied high and you get full speed only.

The L298N has an onboard 5V regulator and a jumper next to the terminals. With that jumper on and a supply under about 12V, the +5V pin outputs 5V and you *could* power the ESP32 from it. Don't, while learning: a motor stall will drag that rail down and reset your board. Keep the ESP32 on USB.

**Never wire a motor to a GPIO pin.** A pin can source about 12mA. A small TT motor draws 200mA running and over 1A stalled. You will destroy the pin instantly. This is not a "probably fine" situation.

How it works

An H-bridge is four electronic switches arranged in an H with the motor in the middle. Close the top-left and bottom-right and current flows one way; close the other diagonal and it flows the other way, so the motor reverses. Closing both switches on the same side is a dead short across the battery — good drivers have logic preventing that, which is why you buy a module rather than build it from four transistors. Full picture in [[H-Bridge motor driver]] and [[Transistors and MOSFETs]].

|IN1|IN2|Result|
|---|---|---|
|HIGH|LOW|Forward|
|LOW|HIGH|Reverse|
|LOW|LOW|Coast (motor freewheels)|
|HIGH|HIGH|Brake (windings shorted, resists turning)|

ENA is separate and takes PWM. You're not lowering the voltage — you're switching full voltage on and off thousands of times a second and the motor's own inertia averages it out. Same trick as the LED dimmer, different scale. See [[PWM output (ESP32)]].

The reason a motor can't touch a GPIO isn't only current. A motor is a coil, and a coil hates having its current interrupted — when you switch it off it generates a large reverse voltage spike trying to keep the current flowing. That spike will happily punch through a microcontroller. The L298N has protection diodes built in to catch it. That's [[Flyback diode protection]], and the general behaviour is [[DC Motors]].

One more real-world note: below about 20-30% duty the motor will hum but not turn. There isn't enough torque to overcome static friction. That deadband matters a lot once you're doing [[Feedback and PID control]] on a robot. See also [[Torque and Gearing]] and [[Batteries and Power budgets]].

Code

```cpp
/*
1. IN1/IN2 set direction, ENA takes a PWM duty for speed
2. setMotor() takes -255..255 so one number carries both
3. never flip direction at full speed, ramp down through zero first
4. the pot's centre position is a deadzone so the motor can actually stop
5. a stall detector isn't here, but it's the obvious next step
*/

const int ENA = 14;    // PWM speed pin
const int IN1 = 27;
const int IN2 = 26;
const int POT = 34;

const int PWM_FREQ = 1000;   // 1kHz suits brushed motors
const int PWM_BITS = 8;      // 0..255

int currentSpeed = 0;

// speed from -255 (full reverse) through 0 (stop) to +255 (full forward)
void setMotor(int speed) {
  speed = constrain(speed, -255, 255);

  if (speed > 0) {
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
  } else if (speed < 0) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
  } else {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);   // coast
  }

  ledcWrite(ENA, abs(speed));
  currentSpeed = speed;
}

// ease towards a target so we never slam from +255 to -255
void rampTo(int target) {
  int stepSize = (target > currentSpeed) ? 5 : -5;

  while (abs(target - currentSpeed) > 5) {
    setMotor(currentSpeed + stepSize);
    delay(10);
  }
  setMotor(target);
}

void setup() {
  Serial.begin(115200);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  ledcAttach(ENA, PWM_FREQ, PWM_BITS);   // core 3.x API

  setMotor(0);

  // quick self test on boot
  Serial.println("forward");  rampTo(200);  delay(1500);
  Serial.println("stop");     rampTo(0);    delay(500);
  Serial.println("reverse");  rampTo(-200); delay(1500);
  Serial.println("stop");     rampTo(0);
  Serial.println("knob control now live");
}

void loop() {
  int raw = analogRead(POT);                    // 0..4095
  int target = map(raw, 0, 4095, -255, 255);    // centre knob = stop

  // deadzone: the motor won't move below ~60 anyway, so call it zero
  if (abs(target) < 60) target = 0;

  if (abs(target - currentSpeed) > 10) {
    setMotor(target);
    Serial.printf("speed %d\n", target);
  }

  delay(20);
}
```

Make it better

- Add the second motor channel (ENB/IN3/IN4) — that's a two-wheeled chassis, and the start of [[Line following robot]]
- Add an encoder (or just a slotted disc and an IR sensor) to measure actual RPM
- Close the loop: PID the PWM until measured RPM matches a target, per [[Feedback and PID control]]
- Detect a stall by watching for commanded speed with no measured rotation, and cut power before something burns
- Measure your battery's sag under load with a multimeter — [[Batteries and Power budgets]]
