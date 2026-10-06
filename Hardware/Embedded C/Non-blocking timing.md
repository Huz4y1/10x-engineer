delay() stops your entire program dead, so the moment you need two things happening at once it has to go.

Why delay ruins everything

```cpp
/*
1. this blinks correctly
2. and it cannot read the button, because for 1998 of every 2000
   milliseconds the CPU is sitting inside delay() doing nothing
3. a press during the delay is simply never seen
*/

void loop() {
    digitalWrite(LED, HIGH);
    delay(1000);                   // 1 second of total deafness
    digitalWrite(LED, LOW);
    delay(1000);

    if (digitalRead(BUTTON) == LOW) {   // checked twice a second, at best
        doSomething();
    }
}
```

```
  blocking

  ├─LED on─┤████ delay 1000 ████├─LED off─┤████ delay 1000 ████├
            ▲            ▲
            └─ button pressed here, and here, both missed


  non-blocking

  ├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤├─┤
   loop runs thousands of times a second, checks the clock, and
   only acts when it's time. Nothing is ever missed.
```

The millis pattern

```cpp
/*
1. millis() counts milliseconds since boot and never blocks
2. remember when the thing last happened
3. every pass through loop, ask "has enough time gone by yet"
4. if yes, update the timestamp and do the work
*/

#define LED 13

unsigned long lastBlink = 0;
const unsigned long INTERVAL = 1000;

bool ledState = false;

void setup() {
    pinMode(LED, OUTPUT);
}

void loop() {
    unsigned long now = millis();

    if (now - lastBlink >= INTERVAL) {
        lastBlink = now;

        ledState = !ledState;
        digitalWrite(LED, ledState);
    }

    // loop is free the rest of the time, so this actually works
    if (digitalRead(BUTTON) == LOW) {
        doSomething();
    }
}
```

Several timed things at once

```cpp
/*
1. one timestamp per job, each with its own interval
2. they don't interfere, and nothing waits for anything else
3. this is a cooperative scheduler in about ten lines
*/

unsigned long lastBlink  = 0;
unsigned long lastSensor = 0;
unsigned long lastUpload = 0;

void loop() {
    unsigned long now = millis();

    if (now - lastBlink >= 500) {          // 2 Hz
        lastBlink = now;
        digitalWrite(LED, !digitalRead(LED));
    }

    if (now - lastSensor >= 100) {         // 10 Hz
        lastSensor = now;
        temperature = readSensor();
    }

    if (now - lastUpload >= 60000) {       // once a minute
        lastUpload = now;
        sendToServer(temperature);
    }

    handleButton();                        // and this runs every single pass
}
```

Tidier, as a table

```cpp
/*
1. a struct per job keeps the state and the config together
2. adding a fourth job is one line, not three variables
3. this is how a real cooperative scheduler starts
*/

typedef struct {
    unsigned long interval;
    unsigned long last;
    void (*run)(void);
} Task;

Task tasks[] = {
    { 500,   0, blink  },
    { 100,   0, sample },
    { 60000, 0, upload },
};

void loop() {
    unsigned long now = millis();

    for (int i = 0; i < sizeof(tasks) / sizeof(tasks[0]); i++) {
        if (now - tasks[i].last >= tasks[i].interval) {
            tasks[i].last = now;
            tasks[i].run();
        }
    }
}
```

The subtraction trick, and why it matters

```c
/*
1. millis() is a uint32_t, it wraps to 0 after about 49.7 days
2. UNSIGNED subtraction wraps too, so the difference stays correct
3. this is why you write (now - last >= interval) and never (now >= last + interval)
*/

// correct, survives the rollover
if (now - lastBlink >= INTERVAL) { }

// BROKEN once millis wraps
if (now >= lastBlink + INTERVAL) { }

// worked example at the wrap point
// last = 4294967000   (near the top of uint32_t)
// now  =         200  (just wrapped)
// now - last = 200 - 4294967000 = 496  in unsigned maths. Correct.
```

|Function|Unit|Wraps after|
|---|---|---|
|`millis()`|milliseconds|~49.7 days|
|`micros()`|microseconds|~71 minutes|

Timing things, and measuring how long they took

```cpp
unsigned long start = micros();

readSensor();

Serial.println(micros() - start);    // output: 1420    microseconds
```

When delay is actually fine

|Situation|Verdict|
|---|---|
|A quick test sketch|Fine|
|In `setup()`, waiting for a sensor to power up|Fine|
|A few microseconds for a protocol|Fine, use `delayMicroseconds()`|
|Anywhere in `loop()` in real code|No|
|Inside an ISR|Never, it hangs the board|

```
  Rule of thumb:
    "wait a bit"                      -> millis pattern
    "wait exactly, and never drift"   -> [[Timers (ESP32)]]
    "several jobs interleaved"        -> the table above
    "it's getting complicated"        -> [[State machines in embedded C]]
```

On the ESP32 specifically, delay does yield

```cpp
/*
1. Arduino's delay() on an ESP32 calls vTaskDelay underneath
2. so FreeRTOS, WiFi and the watchdog all keep running during it
3. YOUR loop still stops though, so the argument above stands
4. a busy-wait, however, will starve other tasks and trip the watchdog
*/

delay(1000);                  // yields to the OS, other tasks run

while (millis() - t < 1000);  // busy-wait, hogs the core, watchdog reboot
```
