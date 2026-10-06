A mechanical button does not go cleanly from off to on, the metal contacts physically bounce apart and back together several times before settling, and your ESP32 is fast enough to see every one of them.

Why one press reads as five

The contacts inside a tactile switch are springy metal. When they slam together they rebound, like dropping a ball. For somewhere between 1ms and 20ms the connection chatters open and closed.

```
   What you think happens:

   HIGH ────────┐
                │
   LOW          └──────────────────────────


   What actually happens:

   HIGH ────────┐ ┌┐ ┌──┐
                │ ││ │  │
   LOW          └─┘└─┘  └──────────────────
                |<-- ~5ms -->|
```

An ESP32 at 240MHz runs `loop()` maybe a hundred thousand times in that 5ms window. So a counter increments by 4, a toggle ends up back where it started, a menu jumps three items. The classic symptom is an LED toggle that works about half the time, seemingly at random.

Both fixes do the same thing: ignore changes that happen too soon after the last one.

The software fix

This is the one to use. It costs nothing and it is easy to tune.

```cpp
#define BTN_PIN 4
#define LED_PIN 2

const unsigned long DEBOUNCE_MS = 50;

int stableState  = HIGH;      // the state we trust
int lastReading  = HIGH;      // the raw pin, may be bouncing
unsigned long lastChange = 0;
bool ledOn = false;

void setup() {
  Serial.begin(115200);
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int reading = digitalRead(BTN_PIN);

  // any change at all restarts the timer
  if (reading != lastReading) {
    lastChange = millis();
    lastReading = reading;
  }

  // only accept it once it has held still long enough
  if (millis() - lastChange > DEBOUNCE_MS) {
    if (reading != stableState) {
      stableState = reading;

      if (stableState == LOW) {         // pressed
        ledOn = !ledOn;
        digitalWrite(LED_PIN, ledOn);
        Serial.println("press");
      }
    }
  }
}
```

The logic in one sentence: every time the raw pin changes, restart a 50ms timer, and only believe the reading once the timer has expired without further changes.

50ms is a good default. Bounce is typically under 20ms, and 50ms is far below the ~150ms at which a human notices lag. If you still get double triggers on a worn switch, go to 80ms. If you need fast repeated presses, try 20ms.

Note that the debounce does not use `delay()`. Using `delay(50)` after a press also "works" but freezes your whole program for 50ms every press, and it does nothing about the bounce on release.

The hardware fix

An RC low-pass filter. The capacitor cannot change voltage instantly, so brief chatter gets smoothed into a slow ramp that crosses the logic threshold once.

```
              3V3
               │
            [10kΩ]
               │
               ├───[1kΩ]───┬────── ESP32 GPIO4
               │           │
             [BTN]       ─┴─ 100nF
               │         ─┬─
              GND          │
                          GND
```

|From|To|Note|
|---|---|---|
|ESP32 3V3|10kΩ resistor leg 1|pull-up|
|10kΩ resistor leg 2|Button leg 1|and to 1kΩ resistor leg 1|
|Button leg 2|ESP32 GND|opposite pair of the tactile switch|
|10kΩ / button node|1kΩ resistor leg 1|series resistor into the filter|
|1kΩ resistor leg 2|ESP32 GPIO4|and to capacitor leg 1|
|Capacitor leg 1|Same node as GPIO4|100nF ceramic, no polarity|
|Capacitor leg 2|ESP32 GND|-|

The time constant is what sets the smoothing:

```
τ = R × C
  = 1000Ω × 0.0000001F
  = 0.0001s
  = 0.1ms
```

0.1ms is too fast to help. For real debouncing you want the capacitor to take a few milliseconds to charge, so use the 10kΩ as the charging resistor and a bigger capacitor:

```
τ = 10000 × 0.0000001
  = 1ms
```

Roughly 1ms, and the pin needs several time constants to cross the threshold, so this smooths a few milliseconds of chatter. For a badly bouncing switch use 1µF instead of 100nF and you get 10ms.

Side by side

|  |Software|Hardware RC|
|---|---|---|
|Cost|Free|Two resistors, one capacitor|
|Board space|None|Three components per button|
|Tunable|Change one number, reflash|Desolder and swap parts|
|Ten buttons|Same code, no extra parts|Thirty extra components|
|Works with interrupts|Needs care, the ISR still fires on every bounce|Yes, the interrupt only fires once|
|CPU cost|A few instructions per loop|None|
|Verdict|**Use this**|Only when you need clean interrupts or the switch is truly awful|

Software wins almost every time. The one genuine case for hardware is when the button drives an interrupt and you cannot afford the ISR to fire twenty times.

Debouncing an interrupt in software

If you do use an interrupt, debounce inside the handler by ignoring anything too soon after the last accepted event.

```cpp
#define BTN_PIN 4

volatile bool pressEvent = false;
volatile unsigned long lastAccepted = 0;

void IRAM_ATTR onPress() {
  unsigned long now = millis();
  if (now - lastAccepted > 50) {
    lastAccepted = now;
    pressEvent = true;
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(BTN_PIN, INPUT_PULLUP);
  attachInterrupt(BTN_PIN, onPress, FALLING);
}

void loop() {
  if (pressEvent) {
    pressEvent = false;
    Serial.println("press");
  }
}
```

`millis()` is safe to call inside an ESP32 ISR. Keep the handler this short, set a flag and leave. Do not print, do not do WiFi, do not use `delay()` in an interrupt handler.

Reusable version

Wrap it up once and stop thinking about it.

```cpp
class Button {
  private:
    uint8_t pin;
    unsigned long debounceMs;
    int stableState;
    int lastReading;
    unsigned long lastChange;

  public:
    Button(uint8_t p, unsigned long d = 50) {
      pin = p;
      debounceMs = d;
      stableState = HIGH;
      lastReading = HIGH;
      lastChange = 0;
    }

    void begin() {
      pinMode(pin, INPUT_PULLUP);
    }

    // returns true once, on the moment of pressing
    bool wasPressed() {
      int reading = digitalRead(pin);

      if (reading != lastReading) {
        lastChange = millis();
        lastReading = reading;
      }

      if (millis() - lastChange > debounceMs) {
        if (reading != stableState) {
          stableState = reading;
          if (stableState == LOW) return true;
        }
      }
      return false;
    }
};


Button btn(4);

void setup() {
  Serial.begin(115200);
  btn.begin();
}

void loop() {
  if (btn.wasPressed()) {
    Serial.println("press");
  }
}
```

The Bounce2 library does the same job if you would rather not maintain it yourself.

What is not bounce

Not every misbehaving button is a bounce problem. Check these first:

|Symptom|Actually is|
|---|---|
|Triggers when you wave your hand near it|Floating pin, no pull-up, see [[Button with Pull-up and Pull-down]]|
|Reads pressed permanently|Two legs from the same side of the tactile switch|
|Triggers when a motor starts|Electrical noise on the supply, needs [[Decoupling capacitors and clean power]]|
|Nothing at all|Wrong pin, or GPIO34-39 with no external pull-up|
|Fires 3-5 times per press|Genuine bounce, debounce it|

The mistake everyone makes

Using `delay(200)` after detecting a press. It appears to fix it, and it does stop the multiple triggers, but it also freezes everything else in the program every time anyone touches a button. Add a second button and they start blocking each other. Use the `millis()` pattern.

Second: debouncing only the press and not the release. If your code reacts to both edges, the release bounces too.

Third: assuming a floating pin is a bounce problem. Debouncing a pin with no pull-up will not fix it, because the signal is not bouncing, it is picking up noise.
