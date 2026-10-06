A switch is just a piece of wire you can break and rejoin, everything confusing about them comes from how the legs are arranged inside the plastic.

Momentary vs latching

|  |Momentary|Latching|
|---|---|---|
|Behaviour|Connected only while held|Stays where you put it|
|Example|Tactile pushbutton, keyboard key|Toggle switch, slide switch, rocker|
|Returns by|Internal spring|Nothing, it stays|
|Use for|Buttons, triggers, resets|Power on/off, mode select|
|Reading it|Watch for the change (edge)|Just read the level|

The little square 6mm tactile buttons in every starter kit are momentary. The tiny slide switches are latching.

Schematic symbol

```
  3V3 ───o  o─── to pin        open (not pressed)

  3V3 ───o──o─── to pin        closed (pressed)
```

Usually drawn as `[BTN]` inline when the detail does not matter.

The 4-pin tactile switch, which confuses everyone

A 4-pin tactile switch does not have four independent contacts. It has two, each brought out to two legs on opposite sides. The two legs on the same side are **permanently connected to each other**, button or no button. Pressing it joins the two pairs.

```
        1 ┌───────────┐ 2
          │           │
          │  ┌─────┐  │      1 and 2  : always connected
          │  │ BTN │  │      3 and 4  : always connected
          │  └─────┘  │      press    : joins the two pairs
          │           │
        3 └───────────┘ 4
```

Same thing drawn as the internal wiring:

```
   1 ────────────┬──────────── 2
                 │
                 o     <- the actual contact, closed when pressed
                 │
   3 ────────────┴──────────── 4
```

So the rule is: use one leg from the top row and one from the bottom row. 1 and 3, or 1 and 4, or 2 and 3. Never 1 and 2.

Why this bites you on a breadboard: the switch straddles the centre channel, and the legs on the left are one pair while the legs on the right are the other pair... or the other way round, depending on which way you rotated it 90°. Get it wrong and the button appears to be permanently pressed, because you wired to two legs that were already joined.

The fix takes ten seconds: multimeter on continuity beep, touch the two legs you plan to use, and it should be silent until you press. If it beeps with the button released, rotate the switch 90° in the breadboard.

Which way to orient it

Push it in so the legs straddle the centre channel of the breadboard. The pairs run left-right in that orientation, so the leg on the left of the channel and the leg on the right of the channel are the two you want. If it does not want to sit that way, turn it 90° and it will.

Other switch types you will meet

|Name|Means|Legs|Example|
|---|---|---|---|
|SPST|Single pole, single throw|2|Simple on/off|
|SPDT|Single pole, double throw|3|Common + two positions, a selector|
|DPDT|Double pole, double throw|6|Two SPDT switches on one lever|
|NO|Normally open|-|Off until acted on, standard button|
|NC|Normally closed|-|On until acted on, e.g. a door sensor|

"Pole" is how many separate circuits it switches. "Throw" is how many positions each one can go to.

Wiring one to an ESP32

Do not wire a button between 3V3 and a pin with nothing else. That gives you a floating pin when released, which reads random garbage. The standard, cheapest, works-every-time arrangement is button to GND with the ESP32's internal pull-up turned on.

```
  3V3 (internal pull-up inside the chip)
        │
        ├───── ESP32 GPIO4
        │
      [BTN]
        │
       GND
```

|From|To|Note|
|---|---|---|
|ESP32 GPIO4|Button leg 1|input pin with INPUT_PULLUP|
|Button leg 2|ESP32 GND|the opposite pair, not the same side|
|-|-|no external resistor needed|

```cpp
#define BTN_PIN 4

void setup() {
  Serial.begin(115200);
  pinMode(BTN_PIN, INPUT_PULLUP);   // idle reads HIGH
}

void loop() {
  // inverted logic: LOW means pressed, because the button pulls to GND
  if (digitalRead(BTN_PIN) == LOW) {
    Serial.println("pressed");
  }
  delay(50);
}
```

The inverted logic feels wrong the first time. Idle = HIGH, pressed = LOW. Get used to it, this arrangement is used everywhere because it is the most noise-resistant.

ESP32 pin warnings

|Pin|Problem|
|---|---|
|GPIO34-39|Input only, and **no internal pull-up or pull-down at all**|
|GPIO0|Boot mode pin, held low at reset = flash mode|
|GPIO12|Wrong level at boot can stop the board starting|
|GPIO6-11|Wired to the internal flash chip, do not touch|

If you must use GPIO34-39 for a button, you have to add a real 10kΩ resistor externally. That catches people out constantly, because the code is identical and just silently does not work.

The mistake everyone makes

Using two legs from the same side of a tactile switch, so the circuit is permanently closed and the input never changes. Symptom: your button reads "pressed" forever, or reads "not pressed" forever depending on which way round. Rotate the switch 90° before you debug anything else.

Second mistake: expecting one press to produce one event. It will not. A mechanical contact bounces for a few milliseconds and your ESP32 samples it thousands of times during that. See [[Button debouncing]].
