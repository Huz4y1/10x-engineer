A hardware timer counts clock ticks in the background and interrupts you when it hits a target, so timing keeps working no matter what loop() is doing.

The four timers

|Timer|Notes|
|---|---|
|0, 1|Group 0. Timer 0 is often used by the task watchdog|
|2, 3|Group 1. Usually free|

Each counts up from a prescaled 80 MHz clock.

Setting one up

```cpp
/*
1. timerBegin(number, prescaler, countUp)
2. 80 MHz / 80 = 1 MHz, so one tick = 1 microsecond. Always use 80, it's easy to reason about
3. timerAlarmWrite sets the target in ticks, true = auto-reload and repeat
4. the callback follows every ISR rule from [[Interrupts (ESP32)]]
*/

hw_timer_t *timer = NULL;

volatile bool tick = false;

void IRAM_ATTR onTimer() {
    tick = true;                // flag only, no printing
}

void setup() {
    Serial.begin(115200);

    timer = timerBegin(0, 80, true);              // timer 0, 1 tick = 1us
    timerAttachInterrupt(timer, &onTimer, true);  // true = edge triggered
    timerAlarmWrite(timer, 1000000, true);        // 1,000,000 us = 1 second
    timerAlarmEnable(timer);
}

void loop() {
    if (tick) {
        tick = false;
        Serial.println("one second");
    }
}

// output: one second
//         one second
```

Working out the alarm value

```
  tick period = prescaler / 80,000,000

  prescaler 80    -> 1 tick = 1 us       ← use this
  prescaler 8000  -> 1 tick = 100 us
  prescaler 80    and alarm 500          -> 500 us   (2 kHz)
  prescaler 80    and alarm 1000         -> 1 ms     (1 kHz)
  prescaler 80    and alarm 1000000      -> 1 s

  The prescaler is 16-bit, so 1 to 65535.
```

A precise sampling loop, which millis can't do

```cpp
/*
1. sample an ADC at exactly 1 kHz regardless of what else is running
2. the timer fills a buffer, loop() reads it out when full
3. this is the reason hardware timers exist
*/

hw_timer_t *timer = NULL;

volatile uint16_t buffer[256];
volatile int      index = 0;
volatile bool     full  = false;

void IRAM_ATTR sample() {
    if (index < 256) {
        buffer[index++] = analogRead(32);
    } else {
        full = true;
    }
}

void setup() {
    Serial.begin(115200);
    timer = timerBegin(1, 80, true);
    timerAttachInterrupt(timer, &sample, true);
    timerAlarmWrite(timer, 1000, true);      // 1000 us = 1 kHz
    timerAlarmEnable(timer);
}

void loop() {
    if (full) {
        timerAlarmDisable(timer);

        for (int i = 0; i < 256; i++) {
            Serial.println(buffer[i]);
        }

        index = 0;
        full  = false;
        timerAlarmEnable(timer);
    }
}
```

Stopping, restarting, reading

```cpp
timerAlarmDisable(timer);      // stop firing, keep the config
timerAlarmEnable(timer);       // start again
timerWrite(timer, 0);          // reset the count to 0
timerEnd(timer);               // free the timer entirely

uint64_t ticks = timerRead(timer);   // current count
```

Timers versus millis

|                |`millis()`|Hardware timer|
|---|---|---|
|Accuracy|Drifts if `loop()` is slow or blocked|Exact, hardware counts independently|
|Shortest interval|~1 ms|Sub-microsecond|
|Blocked by a long `loop()`|Yes|No|
|Blocked by `delay()`|Yes|No|
|Complexity|Trivial|ISR rules apply|
|Use for|Blinking, polling sensors, UI|Sampling, motor control, protocol timing|

```
  Rule of thumb:
    "roughly every second"        -> millis(), see [[Non-blocking timing]]
    "exactly every 500us"         -> hardware timer
    "and it must not drift"       -> hardware timer
```

A watchdog, the other thing timers are for

```cpp
/*
1. a watchdog reboots the board if your code stops feeding it
2. useful for something unattended that occasionally locks up
3. feed it inside loop(), and if loop() ever hangs, the board restarts
*/

#include <esp_task_wdt.h>

void setup() {
    esp_task_wdt_init(5, true);      // 5 second timeout, true = panic and reboot
    esp_task_wdt_add(NULL);          // watch the current task (loop)
}

void loop() {
    esp_task_wdt_reset();            // "I'm still alive"
    doWork();
}
```

The gotcha

```
  A timer callback is an ISR. No Serial.print, no delay, no String,
  no malloc. Mark it IRAM_ATTR. Set a flag and get out.

  timerAlarmWrite takes MICROSECONDS when the prescaler is 80,
  not milliseconds. Passing 1000 thinking it's a second gives you
  a 1 kHz interrupt storm and the board falls over.

  Timer 0 may already be taken by the task watchdog. If timer 0
  behaves oddly, use 2 or 3.
```
