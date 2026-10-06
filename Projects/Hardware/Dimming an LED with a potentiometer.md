Turn a knob, the LED gets brighter. Your first analog input and your first analog-ish output.

What you'll learn

- What an ADC does and what "12-bit" means in practice
- That a potentiometer is just an adjustable [[Voltage dividers]]
- PWM — faking analog output by switching very fast
- Mapping one number range onto another

Parts

|Part|Qty|Rough price|Already own?|
|---|---|---|---|
|ESP32 dev board|1|£8|Yes|
|Breadboard + wires|1|£7|Yes|
|5mm LED|1|£0.10|Yes|
|220Ω resistor|1|£0.05|Yes|
|10kΩ potentiometer|1|£0.60|Probably not|

Any value from 1kΩ to 100kΩ works. 10kΩ is the standard choice — low enough to be immune to noise, high enough not to waste current.

Wiring

```
  3V3 ─────────────┐
                   │
              ┌────┴────┐
              │ [10kΩ]  │  potentiometer
              │  pot    │
              └─┬─────┬─┘
                │     │
             wiper    └───── GND
                │
        ESP32   │
     ┌──────────┴──┐
     │  GPIO34     │  analog in (input only pin)
     │             │
     │  GPIO2      ├───[220Ω]───▶|─── GND
     │             │              LED
     │  GND        ├────────────────── GND
     └─────────────┘
```

|From|To|Note|
|---|---|---|
|Pot outer pin 1|ESP32 3V3|top of the divider|
|Pot outer pin 3|ESP32 GND|bottom of the divider|
|Pot middle pin (wiper)|ESP32 GPIO34|0-3.3V depending on knob position|
|ESP32 GPIO2|Resistor leg 1|PWM output|
|Resistor leg 2|LED anode (long leg)|220Ω limits current|
|LED cathode (short leg)|ESP32 GND|completes the circuit|

Wire the pot to **3V3, not 5V**. The ESP32's ADC pins are 3.3V maximum — feeding one 5V will damage the pin, permanently, and possibly the chip. If a pot outer pin ever touches 5V, that's the mistake to look for.

Also: use an ADC1 pin (GPIO32-39). ADC2 pins stop working the moment WiFi is on, which is a genuinely infuriating bug to discover later. GPIO34-39 are input-only, which is fine here.

How it works

A potentiometer is a strip of resistive material with a slider on it. The two outer pins are the whole strip; the middle pin is wherever the slider is sitting. Turn the knob and you're moving the tap point on a voltage divider, so the wiper voltage slides smoothly from 0V to 3.3V. Full theory in [[Voltage dividers]].

The ADC (analog-to-digital converter) measures that voltage and gives you a number. The ESP32's is 12-bit, so 0-4095: 0 means 0V, 4095 means ~3.3V. It is not a precision instrument — it's noisy and not quite linear at the extremes. See [[Analog input and ADC (ESP32)]] and [[Noise and Filtering]].

For output, the ESP32 can't produce an actual variable voltage. Instead it uses PWM: it switches the pin fully on and fully off thousands of times a second and varies the ratio. 25% on time = a quarter of the average power = a dimmer LED. Your eye can't follow 5kHz so it just looks dim. Details in [[PWM output (ESP32)]].

The chain:

```mermaid
flowchart LR
  A[Knob position] --> B[Voltage 0-3.3V] --> C[ADC reads 0-4095] --> D[Map to 0-255] --> E[PWM duty] --> F[LED brightness]
```

Code

```cpp
/*
1. ledcAttach sets up a hardware PWM channel: pin, frequency, resolution
2. analogRead gives 0-4095 from the 12-bit ADC
3. map() rescales that to the 0-255 our 8-bit PWM wants
4. a rolling average smooths out ADC noise so the LED doesn't flicker
5. ledcWrite sets the duty cycle
*/

const int POT_PIN = 34;    // ADC1, safe to use alongside WiFi
const int LED_PIN = 2;

const int PWM_FREQ = 5000; // 5kHz, well above what the eye can see
const int PWM_BITS = 8;    // 8-bit resolution -> duty 0..255

int smoothed = 0;          // running average of the ADC

void setup() {
  Serial.begin(115200);

  // newer ESP32 Arduino core (3.x): pin, frequency, resolution
  ledcAttach(LED_PIN, PWM_FREQ, PWM_BITS);

  // on core 2.x it's the older three-call form instead:
  //   ledcSetup(0, PWM_FREQ, PWM_BITS);
  //   ledcAttachPin(LED_PIN, 0);
  //   ledcWrite(0, duty);

  smoothed = analogRead(POT_PIN);
}

void loop() {
  int raw = analogRead(POT_PIN);            // 0 .. 4095

  // cheap low-pass filter: 80% old value, 20% new reading
  smoothed = (smoothed * 4 + raw) / 5;

  int duty = map(smoothed, 0, 4095, 0, 255); // rescale to PWM range
  ledcWrite(LED_PIN, duty);

  static unsigned long lastPrint = 0;
  if (millis() - lastPrint > 200) {          // don't flood the serial monitor
    lastPrint = millis();
    Serial.printf("raw %4d  duty %3d\n", raw, duty);
  }
}
```

You'll notice the LED looks like it jumps to full brightness in the first third of the knob and barely changes after. That's not a bug — your eye responds logarithmically, so a linear duty cycle doesn't look linear.

Make it better

- Fix the perceived brightness by cubing the value: `duty = (v*v*v) / (255*255)`
- Drive three LEDs from one knob so it fades red → yellow → green
- Replace the pot with an LDR (light sensor) so the LED brightens as the room darkens
- Print the actual voltage: `raw * 3.3 / 4095` and check it against a multimeter
