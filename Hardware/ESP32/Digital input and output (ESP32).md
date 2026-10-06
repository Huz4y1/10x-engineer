Digital means the pin is only ever fully on (3.3V) or fully off (0V), nothing in between.

Turning a pin on and off

```cpp
/*
1. pinMode tells the chip whether the pin drives or listens
2. digitalWrite HIGH puts 3.3V on it, LOW puts 0V
3. this is the whole of digital output
*/

#define LED 13

void setup() {
    pinMode(LED, OUTPUT);
}

void loop() {
    digitalWrite(LED, HIGH);   // 3.3V, LED on
    delay(1000);
    digitalWrite(LED, LOW);    // 0V, LED off
    delay(1000);
}
```

```
  wiring an LED, the resistor is not optional

   GPIO 13 ──[220R]──▶|── GND
                     LED
                    long leg
                    to resistor

  Without the resistor the LED pulls far more than 40mA
  and takes the pin with it.
```

Reading a pin

```cpp
/*
1. a bare INPUT pin with nothing attached FLOATS, it reads random noise
2. INPUT_PULLUP switches an internal resistor to 3.3V so it idles HIGH
3. so the button connects the pin to GND, and pressed reads LOW
4. this feels backwards and that is normal
*/

#define BUTTON 4
#define LED    13

void setup() {
    Serial.begin(115200);
    pinMode(BUTTON, INPUT_PULLUP);
    pinMode(LED, OUTPUT);
}

void loop() {
    int state = digitalRead(BUTTON);

    if (state == LOW) {            // LOW means pressed here
        digitalWrite(LED, HIGH);
    } else {
        digitalWrite(LED, LOW);
    }
}

// output: nothing printed, but the LED follows the button
```

```
  button with the internal pull-up, 2 wires only

           3V3
            │
           ┌┴┐  internal 45k pull-up, inside the chip
           └┬┘
   GPIO 4 ──┤
            │
           ─┴─  button
            │
           GND

   not pressed  ->  pin sits at 3V3  ->  digitalRead = HIGH
   pressed      ->  pin pulled to 0  ->  digitalRead = LOW
```

The pin modes

|Mode|What it does|
|---|---|
|`OUTPUT`|Pin drives HIGH or LOW|
|`INPUT`|Pin listens, floats if nothing is attached|
|`INPUT_PULLUP`|Listens, internal resistor holds it HIGH|
|`INPUT_PULLDOWN`|Listens, internal resistor holds it LOW (ESP32 has this, an Uno doesn't)|
|`OUTPUT_OPEN_DRAIN`|Can only pull LOW, needs an external pull-up|

Reading a change instead of a level

```cpp
/*
1. loop() runs thousands of times a second, so a held button "presses" constantly
2. remember the last state and only act when it changes
3. the small delay is a crude debounce, see [[State machines in embedded C]] for the proper way
*/

#define BUTTON 4

int lastState = HIGH;
int count = 0;

void setup() {
    Serial.begin(115200);
    pinMode(BUTTON, INPUT_PULLUP);
}

void loop() {
    int state = digitalRead(BUTTON);

    if (lastState == HIGH && state == LOW) {   // HIGH -> LOW is the press edge
        count++;
        Serial.println(count);
        delay(30);                             // ride out the contact bounce
    }

    lastState = state;
}

// output: 1
//         2
//         3
```

The ESP-IDF way, meaningfully different

```c
// ESP-IDF, no Arduino layer. Everything is configured through a struct.

#include "driver/gpio.h"

void app_main(void)
{
    gpio_config_t io = {
        .pin_bit_mask = (1ULL << 13),        // a bitmask, so several pins at once
        .mode         = GPIO_MODE_OUTPUT,
        .pull_up_en   = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type    = GPIO_INTR_DISABLE,
    };
    gpio_config(&io);

    while (1) {
        gpio_set_level(13, 1);
        vTaskDelay(pdMS_TO_TICKS(1000));     // never delay(), this yields to FreeRTOS
        gpio_set_level(13, 0);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

The gotcha

```
  Pins 34, 35, 36 and 39 are input only.
  pinMode(34, OUTPUT) compiles, uploads, and does absolutely nothing.
  INPUT_PULLUP on those pins also does nothing.

  And delay() in loop() means nothing else can happen while it waits,
  see [[Non-blocking timing]].
```
