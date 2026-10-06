An HC-SR04 that measures how far away something is by timing an echo. The eyes of every robot you'll build after this.

What you'll learn

- Timing a pulse with microsecond precision
- Why a 5V sensor output needs a divider before it touches an ESP32
- Turning a raw physical measurement into a useful number
- Filtering out garbage readings

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|Breadboard + wires|1|£7|Yes|
|HC-SR04 ultrasonic sensor|1|£2|No|
|1kΩ resistor|1|£0.05|Yes|
|2kΩ resistor (or 2× 1kΩ in series)|1|£0.05|Yes|
|LED + 220Ω (optional proximity alarm)|1|£0.15|Yes|

Wiring

```
  5V (VIN) ────────────────┐
                           │
                    ┌──────┴──────┐
                    │   HC-SR04   │
                    │ VCC TRIG ECHO GND
                    └──┬───┬────┬──┬─┘
                       │   │    │  │
                       │   │    │  └─── GND
                       │   │    │
                       │   │    └──[1kΩ]──┬──── ESP32 GPIO18  (ECHO, divided)
                       │   │              │
                       │   │            [2kΩ]
                       │   │              │
                       │   │             GND
                       │   │
                       │   └──────────────── ESP32 GPIO5  (TRIG)
                       │
                       └──────────────────── ESP32 5V / VIN
```

|From|To|Note|
|---|---|---|
|HC-SR04 VCC|ESP32 5V (VIN)|the sensor genuinely needs 5V|
|HC-SR04 GND|ESP32 GND|common ground|
|HC-SR04 TRIG|ESP32 GPIO5|3.3V out is enough to trigger it|
|HC-SR04 ECHO|1kΩ resistor leg 1|start of the divider|
|1kΩ leg 2|GPIO18 **and** 2kΩ leg 1|this junction is the divided signal|
|2kΩ leg 2|ESP32 GND|bottom of the divider|

**This divider is not optional.** ECHO idles at 0V but pulses to 5V. ESP32 pins are rated to 3.3V. Wiring ECHO straight to a GPIO is the single most common way beginners kill an ESP32 — it might work for a week and then not. The 1kΩ/2kΩ pair drops 5V to 5 × 2/(1+2) = 3.33V. Close enough.

TRIG is fine unprotected: that's the ESP32 talking *to* the sensor, and the HC-SR04 happily accepts 3.3V as a HIGH.

How it works

The sensor is a speaker and a microphone. You pulse TRIG high for 10µs; it emits eight 40kHz chirps (inaudible) and then holds ECHO high for exactly as long as the sound takes to go out and come back.

Sound travels at roughly 343 m/s in air, which is 0.0343 cm per microsecond. The pulse covers the distance twice, so:

`distance_cm = duration_us × 0.0343 / 2`, or just `duration_us / 58`.

Everything about how sensors turn physics into numbers is in [[Sensors overview]], and the divider maths is in [[Voltage dividers]].

Limits worth knowing before you build a robot around it:

|Thing|Reality|
|---|---|
|Range|2cm to about 400cm, reliable to ~200cm|
|Beam|~15° cone, not a laser — it sees wide|
|Soft surfaces|Curtains and jumpers absorb the chirp, reads as nothing|
|Angled surfaces|Sound bounces away, never comes back, reads as nothing|
|Rate|~20 readings/sec max, or echoes from the last ping confuse the next|

Those failure modes all return a very large number or zero, which is why the code filters. See [[Noise and Filtering]].

Code

```cpp
/*
1. TRIG is an output, ECHO is an input
2. a clean 10us HIGH pulse on TRIG starts a measurement
3. pulseIn() blocks and returns how long ECHO stayed HIGH, in microseconds
4. divide by 58 to get centimetres
5. take several readings and use the median so one bad ping can't lie to us
*/

const int TRIG_PIN = 5;
const int ECHO_PIN = 18;
const int LED_PIN  = 2;

const unsigned long TIMEOUT_US = 30000;  // ~5m, gives up rather than hanging

// one raw measurement, returns -1 if nothing came back
float readOnce() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);                  // make sure we start from a clean low
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);                 // the datasheet asks for 10us
  digitalWrite(TRIG_PIN, LOW);

  unsigned long us = pulseIn(ECHO_PIN, HIGH, TIMEOUT_US);
  if (us == 0) return -1.0;              // timed out, nothing in range

  return us / 58.0;                      // microseconds -> cm
}

// median of 5 readings, far more robust than a single ping
float readDistance() {
  float v[5];
  int n = 0;

  for (int i = 0; i < 5; i++) {
    float d = readOnce();
    if (d > 0) v[n++] = d;
    delay(10);                           // let the echoes die down
  }

  if (n == 0) return -1.0;

  // tiny insertion sort, n is at most 5
  for (int i = 1; i < n; i++) {
    float key = v[i];
    int j = i - 1;
    while (j >= 0 && v[j] > key) { v[j + 1] = v[j]; j--; }
    v[j + 1] = key;
  }

  return v[n / 2];
}

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(115200);
}

void loop() {
  float cm = readDistance();

  if (cm < 0) {
    Serial.println("no echo");
    digitalWrite(LED_PIN, LOW);
  } else {
    Serial.printf("%.1f cm\n", cm);
    digitalWrite(LED_PIN, cm < 20);      // light up when something's close
  }

  delay(100);
}
```

Sanity check it against a tape measure at 10cm, 50cm and 100cm before you trust it in a robot.

Make it better

- Replace the LED with a buzzer that beeps faster as things get closer, reversing-sensor style
- Log distance over time and detect movement rather than presence
- Mount it on a servo and sweep it to build a 180° map — that's [[Servo sweep and control]] plus this
- Swap `pulseIn` (which blocks) for an interrupt on ECHO so the CPU stays free — [[Interrupts (ESP32)]]
