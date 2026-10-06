Two wheels, no third point of contact, standing upright by constantly driving into its own fall. An MPU6050 IMU and a real PID loop.

Be honest with yourself first: this is a genuine step up. Everything before this worked the first time if you wired it right. This one will not. You'll spend most of your time tuning three numbers by hand while the robot repeatedly falls over, and that is the actual lesson — control theory only becomes real when you've watched it fail.

What you'll learn

- Reading an accelerometer and gyroscope over I2C
- Sensor fusion — why neither sensor alone gives you an angle
- PID properly: what P, I and D each fix, and how each one fails
- Loop timing, and why a control loop must run at a fixed rate

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|MPU6050 IMU module|1|£3|No|
|L298N (or better, TB6612FNG)|1|£4|Have the L298N|
|N20 gear motors with encoders|2|£14|No|
|Wheels, ~65mm|2|£4|Maybe|
|2S LiPo or 6×AA|1|£10|No|
|Rigid tall chassis|1|£5|No|

The parts choice matters more here than anywhere else so far:

- **TT motors will not work well.** They have huge backlash — a dead zone where the shaft turns but the wheel doesn't. A balancer has to make thousands of tiny corrections and every one of them gets swallowed by that slop. N20 metal-gear motors are much tighter.
- **The L298N is marginal.** Its 2V internal drop and slow switching cost you response speed. A TB6612FNG is £4 and much better.
- **Tall is easier than short.** Counter-intuitive but true — see the physics below.
- **Rigid matters.** A flexible chassis lets the IMU read vibration instead of tilt.

Wiring

```
                    ┌──────────────┐
   3V3 ─────────────┤ VCC   MPU6050│
   GND ─────────────┤ GND          │
   GPIO21 ──────────┤ SDA          │   I2C data
   GPIO22 ──────────┤ SCL          │   I2C clock
                    │ INT ── GPIO19│   optional, data-ready
                    └──────────────┘

        ESP32                       L298N / TB6612
     ┌──────────┐              ┌─────────────┐
     │  GPIO14  ├──────────────┤ ENA         ├── LEFT MOTOR
     │  GPIO27  ├──────────────┤ IN1         │
     │  GPIO26  ├──────────────┤ IN2         │
     │  GPIO32  ├──────────────┤ ENB         ├── RIGHT MOTOR
     │  GPIO25  ├──────────────┤ IN3         │
     │  GPIO33  ├──────────────┤ IN4         │
     │  GND     ├──────────────┤ GND         │
     └──────────┘              │ +12V ── BATTERY
                               └─────────────┘
```

|From|To|Note|
|---|---|---|
|MPU6050 VCC|ESP32 3V3|module has a regulator but 3V3 is safest|
|MPU6050 GND|ESP32 GND|common ground|
|MPU6050 SDA|ESP32 GPIO21|I2C data, default pin|
|MPU6050 SCL|ESP32 GPIO22|I2C clock, default pin|
|MPU6050 AD0|leave floating|address 0x68; tie to 3V3 for 0x69|
|ESP32 GPIO14 / 27 / 26|Driver ENA / IN1 / IN2|left motor|
|ESP32 GPIO32 / 25 / 33|Driver ENB / IN3 / IN4|right motor|
|Driver GND|ESP32 GND|**mandatory**|
|Battery + / −|Driver motor supply|motors only|

Mount the IMU **flat, low, and rigidly**, with its axes square to the robot. A wonky mount adds a fixed angle offset you'll waste an evening chasing. Keep its wires away from the motor wires — motor switching noise couples into I2C and corrupts readings. Short wires, and glue it down.

How it works

An inverted pendulum is unstable: tip it slightly and gravity pulls it further. The only way to stay up is to keep driving the base *underneath* the centre of mass. Fall forward, drive forward. See [[Centre of mass and Stability]].

This is why tall is easier. A tall robot has a larger moment of inertia, so it falls more slowly, so your control loop has more time to react. A short one falls fast and needs a much faster loop. Broom handle on your palm versus a pencil.

**Why you need both sensors.** The MPU6050 has an accelerometer and a gyroscope, and each one is individually useless here:

|Sensor|Gives you|Problem|
|---|---|---|
|Accelerometer|Absolute angle from gravity|Also measures the robot's own acceleration, so it's very noisy while moving|
|Gyroscope|Rate of rotation, very clean|You must integrate it to get an angle, and tiny errors accumulate — it drifts away within seconds|

The accelerometer is right on average but jumpy. The gyro is smooth but slowly wrong. A complementary filter combines them: trust the gyro short-term, let the accelerometer slowly correct the drift.

`angle = 0.98 * (angle + gyroRate * dt) + 0.02 * accelAngle`

That single line is 90% of what a Kalman filter buys you, for 1% of the complexity. More in [[Noise and Filtering]] and [[Sensors overview]]. The bus itself is [[I2C (ESP32)]].

**PID.** The error is `angle - setpoint`. Full theory in [[Feedback and PID control]]; here's what each term actually does on this robot:

|Term|Job|Too low|Too high|
|---|---|---|---|
|P|React proportionally to how far you're tipped|Falls over immediately, no fight|Fast oscillation, buzzes and shakes itself over|
|I|Cancel steady offsets like a wonky IMU or a heavy side|Slowly creeps in one direction and eventually falls|Slow wobble that builds until it falls|
|D|React to how *fast* the angle is changing — damping|Overshoots, oscillates with growing swings|Twitchy, amplifies every bit of sensor noise|

```mermaid
flowchart LR
  I[MPU6050] --> F[Complementary filter] --> A[angle]
  A --> E[error = angle - setpoint] --> P[PID] --> M[motor PWM] --> R[robot tips]
  R --> I
```

Tuning order, and don't skip steps: Kp only until it oscillates around upright, then back it off ~30%. Add Kd until the oscillation damps out. Add a small Ki last, to stop the slow drift. Expect several hours.

**The other reason it won't balance.** Motor deadband. Below ~25% duty the motors hum and don't turn, so all your tiny corrections do nothing and the robot falls before the output gets big enough to act. The code adds a minimum output to jump that gap — see [[Torque and Gearing]] and [[DC Motors]].

Also find the true balance point. It's rarely 0.0°. Hold the robot upright by hand, read the printed angle, and use that as `setpoint`.

Code

```cpp
/*
1. read raw accel and gyro over I2C every loop
2. complementary filter fuses them into one trustworthy angle
3. PID turns angle error into a motor output
4. below MIN_OUTPUT the motors don't move, so we jump the deadband
5. past FALL_LIMIT we give up and cut power rather than thrash
*/

#include <Wire.h>

const int MPU = 0x68;              // I2C address of the MPU6050

const int ENA = 14, IN1 = 27, IN2 = 26;
const int ENB = 32, IN3 = 25, IN4 = 33;

// ---- tune these three, in this order ----
float Kp = 22.0;
float Ki = 0.4;
float Kd = 1.1;

float setpoint   = 0.0;            // your robot's real balance angle
const float FALL_LIMIT = 35.0;     // past this it's on the floor, stop trying
const int   MIN_OUTPUT = 45;       // deadband jump, find yours by experiment

float angle = 0;                   // filtered angle in degrees
float integral = 0;
float lastError = 0;
float gyroBias = 0;                // measured at startup

unsigned long lastLoop = 0;
const unsigned long LOOP_US = 5000;   // 5ms -> 200Hz, must be consistent

void writeReg(uint8_t reg, uint8_t val) {
  Wire.beginTransmission(MPU);
  Wire.write(reg);
  Wire.write(val);
  Wire.endTransmission();
}

// read accel Y/Z and gyro X in one burst
void readIMU(float &accAngle, float &gyroRate) {
  Wire.beginTransmission(MPU);
  Wire.write(0x3B);                       // ACCEL_XOUT_H
  Wire.endTransmission(false);
  Wire.requestFrom(MPU, 14, true);

  int16_t ax = Wire.read() << 8 | Wire.read();
  int16_t ay = Wire.read() << 8 | Wire.read();
  int16_t az = Wire.read() << 8 | Wire.read();
  Wire.read(); Wire.read();               // temperature, ignored
  int16_t gx = Wire.read() << 8 | Wire.read();

  // angle from gravity. swap axes if your IMU is mounted differently
  accAngle = atan2((float)ay, (float)az) * 180.0 / PI;

  // 131 LSB per deg/s at the default +-250 deg/s range
  gyroRate = (gx / 131.0) - gyroBias;
}

// robot must be perfectly still for this
void calibrateGyro() {
  Serial.println("hold still, calibrating gyro");
  float sum = 0;

  for (int i = 0; i < 500; i++) {
    float a, g;
    readIMU(a, g);
    sum += g + gyroBias;                  // undo the bias we're measuring
    delay(3);
  }

  gyroBias = sum / 500.0;
  Serial.printf("gyro bias %.3f deg/s\n", gyroBias);
}

void driveLeft(int s) {
  s = constrain(s, -255, 255);
  digitalWrite(IN1, s >= 0); digitalWrite(IN2, s < 0);
  ledcWrite(ENA, abs(s));
}
void driveRight(int s) {
  s = constrain(s, -255, 255);
  digitalWrite(IN3, s >= 0); digitalWrite(IN4, s < 0);
  ledcWrite(ENB, abs(s));
}

void setup() {
  Serial.begin(115200);
  Wire.begin(21, 22);
  Wire.setClock(400000);                  // fast mode, we need the throughput

  writeReg(0x6B, 0x00);                   // wake the MPU6050 from sleep
  delay(100);
  writeReg(0x1A, 0x03);                   // onboard low-pass filter, 44Hz
  writeReg(0x1B, 0x00);                   // gyro +-250 deg/s
  writeReg(0x1C, 0x00);                   // accel +-2g

  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  ledcAttach(ENA, 20000, 8);              // 20kHz, above hearing
  ledcAttach(ENB, 20000, 8);

  driveLeft(0); driveRight(0);

  calibrateGyro();

  // seed the filter with the accelerometer so we don't start from zero
  float a, g;
  readIMU(a, g);
  angle = a;

  lastLoop = micros();
}

void loop() {
  // fixed rate loop: dt must be constant or the I and D terms lie
  if (micros() - lastLoop < LOOP_US) return;

  float dt = (micros() - lastLoop) / 1000000.0;
  lastLoop = micros();

  float accAngle, gyroRate;
  readIMU(accAngle, gyroRate);

  // complementary filter: gyro for the fast stuff, accel to kill the drift
  angle = 0.98 * (angle + gyroRate * dt) + 0.02 * accAngle;

  // fallen over: stop everything and reset the integral
  if (fabs(angle) > FALL_LIMIT) {
    driveLeft(0); driveRight(0);
    integral = 0;
    return;
  }

  float error = angle - setpoint;

  integral += error * dt;
  integral = constrain(integral, -50, 50);      // anti-windup, essential

  float derivative = (error - lastError) / dt;
  lastError = error;

  float output = Kp * error + Ki * integral + Kd * derivative;
  output = constrain(output, -255, 255);

  // jump the motor deadband: below MIN_OUTPUT nothing turns at all
  int drive = 0;
  if (fabs(output) > 3) {
    drive = (int)output;
    if (drive > 0 && drive < MIN_OUTPUT)  drive = MIN_OUTPUT;
    if (drive < 0 && drive > -MIN_OUTPUT) drive = -MIN_OUTPUT;
  }

  driveLeft(drive);
  driveRight(drive);

  static int n = 0;
  if (++n % 40 == 0) {                          // print at ~5Hz, not 200Hz
    Serial.printf("angle %6.2f  out %5d\n", angle, drive);
  }
}
```

Debug order when it won't balance: (1) print the angle with motors unplugged and check it reads ~0 upright and changes sign correctly when you tip it; (2) check that tipping forward drives the wheels *forward* — if it accelerates the fall, your motor direction or angle sign is inverted, so flip one; (3) only then start tuning.

Make it better

- Add the wheel encoders and a second, slower PID loop on position so it holds its spot instead of wandering
- Add a third loop for heading so it can be steered while balancing
- Add Bluetooth from [[Bluetooth controlled RC car]] and tune Kp/Ki/Kd live from your phone instead of reflashing
- Stream the angle to [[WiFi sensor dashboard]] and tune against a real graph
- Replace the complementary filter with the MPU6050's onboard DMP, or a Kalman filter, and compare
