An interrupt lets the hardware stop whatever your code is doing the instant a pin changes, instead of you constantly asking.

Polling versus interrupting

```
  polling                      interrupt
  ┌──────────┐                 ┌──────────┐
  │ loop     │                 │ loop     │  doing other work
  │  check?  │ no              │          │
  │  check?  │ no              │          │
  │  check?  │ no              │      ────┼──── pin changes
  │  check?  │ YES  ← may      │  ISR     │  runs immediately
  │          │        miss a   │  ────────┼──── back where it was
  └──────────┘        fast     └──────────┘
                      pulse
```

Attaching one

```cpp
/*
1. the ISR is a normal function taking nothing and returning nothing
2. IRAM_ATTR puts it in internal RAM so it still works while flash is busy
3. volatile tells the compiler the variable really does change behind its back
4. the ISR does the bare minimum, loop() does the real work
*/

#define BUTTON 4

volatile bool pressed = false;

void IRAM_ATTR onPress() {
    pressed = true;               // that's it. Nothing else.
}

void setup() {
    Serial.begin(115200);
    pinMode(BUTTON, INPUT_PULLUP);
    attachInterrupt(BUTTON, onPress, FALLING);
}

void loop() {
    if (pressed) {
        pressed = false;
        Serial.println("button!");    // Serial is safe HERE, not in the ISR
    }
}

// output: button!
//         button!
```

The trigger modes

|Mode|Fires when|
|---|---|
|`RISING`|LOW to HIGH|
|`FALLING`|HIGH to LOW|
|`CHANGE`|Either direction|
|`ONLOW`|Continuously while the pin is low|
|`ONHIGH`|Continuously while the pin is high|

```
        ┌──────────┐
   ─────┘          └──────
        ▲          ▲
     RISING     FALLING
        └─ CHANGE ─┘

  A button with INPUT_PULLUP idles HIGH and goes LOW when pressed,
  so FALLING is the press and RISING is the release.
```

The rules for an ISR, and they are hard rules

|Rule|Why|
|---|---|
|Keep it short, microseconds|Everything else is frozen while it runs|
|No `delay()`|It relies on interrupts, which are off. Board hangs|
|No `Serial.print()`|Slow and not interrupt-safe, causes crashes|
|No `malloc` / `new` / `String`|Allocation is not interrupt-safe|
|Mark it `IRAM_ATTR`|Flash may be busy, and then the ISR can't be fetched|
|Shared variables must be `volatile`|See [[Volatile and interrupt safety]]|
|Set a flag, don't do the work|Do the work in `loop()`|

What breaking the rules looks like

```cpp
// DON'T. This is the classic crash.

void IRAM_ATTR bad() {
    Serial.println("pressed");    // not interrupt-safe
    delay(50);                    // needs interrupts, which are disabled
}

// output: Guru Meditation Error: Core 1 panic'ed (Interrupt wdt timeout)
//         and the board reboots forever
```

Counting pulses, where interrupts really earn their keep

```cpp
/*
1. an encoder or flow meter emits pulses far too fast to poll reliably
2. the ISR only increments a counter
3. loop() snapshots the counter with interrupts briefly off, so it can't
   change halfway through the read
*/

#define SENSOR 27

volatile unsigned long pulses = 0;

void IRAM_ATTR onPulse() {
    pulses++;
}

void setup() {
    Serial.begin(115200);
    pinMode(SENSOR, INPUT_PULLUP);
    attachInterrupt(SENSOR, onPulse, FALLING);
}

void loop() {
    noInterrupts();
    unsigned long count = pulses;    // copy it atomically
    pulses = 0;
    interrupts();

    Serial.printf("%lu pulses per second\n", count);
    delay(1000);
}

// output: 0 pulses per second
//         37 pulses per second
```

Debouncing inside the ISR

```cpp
/*
1. a mechanical button bounces, one press can fire the ISR 5-20 times
2. ignore anything arriving within 200ms of the last accepted press
3. millis() is safe in an ISR on the ESP32, delay() is not
*/

volatile unsigned long lastPress = 0;
volatile int count = 0;

void IRAM_ATTR onPress() {
    unsigned long now = millis();

    if (now - lastPress > 200) {
        count++;
        lastPress = now;
    }
}
```

Turning one off

```cpp
detachInterrupt(BUTTON);     // stop it entirely
```

The ESP-IDF way

```c
// ESP-IDF, the handler takes a void* argument so one function can serve many pins

#include "driver/gpio.h"

static void IRAM_ATTR isr_handler(void *arg)
{
    uint32_t pin = (uint32_t) arg;
    // typically: xQueueSendFromISR to hand the work to a task
}

void setup_isr(void)
{
    gpio_set_intr_type(GPIO_NUM_4, GPIO_INTR_NEGEDGE);
    gpio_install_isr_service(0);
    gpio_isr_handler_add(GPIO_NUM_4, isr_handler, (void *) 4);
}
```

The gotcha

```
  Interrupts happen between any two machine instructions, including
  halfway through updating a 64-bit variable or a struct. If loop()
  reads something the ISR writes, and it is bigger than a single
  32-bit word, wrap the read in noInterrupts()/interrupts().

  On an ESP32 an interrupt fires on the core that attached it. An ISR
  on core 1 does not stop core 0.

  If the board reboots with "Interrupt wdt timeout", an ISR is taking
  too long. Almost always a print or a delay inside it.
```
