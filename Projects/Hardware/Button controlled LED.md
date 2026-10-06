A push button that toggles an LED. Your first input.

What you'll learn

- Reading a pin instead of driving it
- Why a floating input reads garbage, and what a pull-up fixes
- What switch bounce is and why it makes toggles misbehave
- Edge detection — reacting to a change, not a state

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|Breadboard + wires|1|£7|Yes|
|5mm LED|1|£0.10|Yes|
|220Ω resistor|1|£0.05|Yes|
|Tactile push button|1|£0.15|Probably not|

Tactile buttons come in packs of 50 for a couple of quid. Get some, you'll use them constantly.

Wiring

```
  3V3
   │
   │        ESP32
   │     ┌─────────┐
   │     │  GPIO2  ├───[220Ω]───▶|─── GND
   │     │         │              LED
   │     │  GPIO4  ├───[BTN]───┐
   │     │         │           │
   │     │  GND    ├───────────┴─── GND
   │     └─────────┘
```

|From|To|Note|
|---|---|---|
|ESP32 GPIO2|Resistor leg 1|LED output|
|Resistor leg 2|LED anode (long leg)|220Ω limits current|
|LED cathode (short leg)|ESP32 GND|completes the LED circuit|
|ESP32 GPIO4|Button pin 1|input pin, internal pull-up on|
|Button pin 2|ESP32 GND|pressing pulls the pin to 0V|

No external resistor on the button — we use the ESP32's internal pull-up. A tactile button has 4 legs but really only 2 connections: the legs are joined in pairs. If it seems permanently pressed, rotate it 90° in the breadboard.

How it works

An input pin with nothing attached is *floating*. It's a tiny antenna picking up mains hum and it will read HIGH and LOW at random. You have to tie it to a known voltage.

`INPUT_PULLUP` switches on a resistor inside the chip (about 45kΩ) between the pin and 3.3V. Left alone the pin reads HIGH. Press the button and you connect the pin straight to GND, which wins against the weak pull-up, so it reads LOW.

So the logic is inverted: pressed = LOW. That trips everyone up once. Detail in [[Button with Pull-up and Pull-down]] and [[Buttons and Switches]].

The other problem is mechanical. The metal contacts physically bounce apart on contact for a millisecond or two, so one press looks like 5-10 presses to a chip reading the pin a million times a second. That's why we only act on a change and then ignore the pin for a moment — see [[Button debouncing]]. General pin behaviour is in [[Digital input and output (ESP32)]].

Code

```cpp
/*
1. INPUT_PULLUP holds the pin HIGH until the button shorts it to GND
2. we remember the previous reading so we can spot the moment it changes
3. HIGH -> LOW is the "just pressed" edge, that's what we act on
4. after an edge we ignore the pin for 50ms to ride out the bounce
5. the LED state is ours to keep, we just flip a bool
*/

const int LED_PIN    = 2;
const int BUTTON_PIN = 4;

bool ledOn = false;             // what we think the LED is doing
int  lastReading = HIGH;        // pull-up means idle is HIGH
unsigned long lastChange = 0;   // when the pin last moved
const unsigned long DEBOUNCE_MS = 50;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);   // no external resistor needed
  Serial.begin(115200);
}

void loop() {
  int reading = digitalRead(BUTTON_PIN);

  // only look at the pin if it has settled for long enough
  if (reading != lastReading && millis() - lastChange > DEBOUNCE_MS) {
    lastChange = millis();

    if (reading == LOW) {              // LOW = pressed, remember it's inverted
      ledOn = !ledOn;                  // flip our state
      digitalWrite(LED_PIN, ledOn);
      Serial.println(ledOn ? "on" : "off");
    }

    lastReading = reading;
  }
}
```

Try commenting out the debounce check. Press the button a few times and watch the LED land on the wrong state — that's bounce, live.

Make it better

- Hold-to-brighten: while held, ramp brightness with [[PWM output (ESP32)]]
- Two buttons, one for on and one for off
- Detect a long press (held > 1s) and do something different
- Move the button to an interrupt so the main loop is free — [[Interrupts (ESP32)]]
