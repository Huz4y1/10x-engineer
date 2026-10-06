There's no debugger attached by default and no console, so debugging embedded code is mostly about making the board tell you what it's doing.

The first question, always

```
  Is it hardware or software?

  ┌─────────────────────────────────────────────┐
  │ Does the board boot at all?                 │
  │   no  -> power, USB, strapping pins         │
  │   yes │                                     │
  ├───────▼─────────────────────────────────────┤
  │ Does a bare blink sketch work?              │
  │   no  -> wrong board selected, bad upload   │
  │   yes │                                     │
  ├───────▼─────────────────────────────────────┤
  │ Does the pin change with a multimeter on it │
  │ when you digitalWrite it?                   │
  │   no  -> software, or a dead pin            │
  │   yes -> the circuit. Wiring, resistor,     │
  │          missing ground                     │
  └─────────────────────────────────────────────┘

  Half of "my code is broken" is a floating ground or a
  breadboard rail that isn't connected end to end.
```

Serial printing, done deliberately

```cpp
/*
1. print the state, not "here" and "here2"
2. include a timestamp so you can see WHEN, not just what
3. print once per change, not every pass, or the output is unreadable
*/

void loop() {
    static int lastState = -1;

    if (state != lastState) {
        Serial.printf("[%lu] state %d -> %d\n", millis(), lastState, state);
        lastState = state;
    }
}

// output: [0] state -1 -> 0
//         [1240] state 0 -> 1
//         [3891] state 1 -> 2
```

Debug prints you can switch off

```cpp
/*
1. a macro that compiles to nothing when DEBUG is off
2. no runtime cost at all in the release build, the code isn't there
3. __LINE__ tells you exactly where it came from
*/

#define DEBUG 1

#if DEBUG
  #define DBG(fmt, ...) Serial.printf("[%lu:%d] " fmt "\n", millis(), __LINE__, ##__VA_ARGS__)
#else
  #define DBG(fmt, ...)
#endif

void loop() {
    int raw = analogRead(32);
    DBG("adc=%d", raw);
}

// output: [1204:32] adc=2048
```

Blink codes, for when there's no serial at all

```cpp
/*
1. useful for a battery device, or once the USB is unplugged
2. N blinks then a long pause, count them
3. crude, and it will tell you what's wrong from across a room
*/

void blinkError(int code) {
    for (;;) {
        for (int i = 0; i < code; i++) {
            digitalWrite(LED, HIGH); delay(200);
            digitalWrite(LED, LOW);  delay(200);
        }
        delay(1500);
    }
}

// 1 = sensor not found
// 2 = wifi failed
// 3 = sd card failed
```

```
  code 2

   ┌─┐ ┌─┐                 ┌─┐ ┌─┐
  ─┘ └─┘ └─────────────────┘ └─┘ └────────────
   ├─────┤   ├───────────┤
   2 blinks     long gap        repeats
```

A heartbeat, the cheapest health check

```cpp
/*
1. one LED blinking steadily from loop()
2. if it stops, loop() has stopped, which is half the diagnosis already
3. costs one pin and four lines
*/

void heartbeat() {
    static unsigned long last = 0;
    if (millis() - last >= 500) {
        last = millis();
        digitalWrite(LED, !digitalRead(LED));
    }
}
```

Reading a crash dump

```
  Guru Meditation Error: Core 1 panic'ed (LoadProhibited)
  Backtrace: 0x400d1234:0x3ffb1f30 0x400d5678:0x3ffb1f50

  LoadProhibited      -> read through a null or bad pointer
  StoreProhibited     -> wrote through a null or bad pointer
  IllegalInstruction  -> jumped somewhere that isn't code,
                         usually a corrupted function pointer
  InstrFetchProhibited-> called a null function pointer
  Interrupt wdt timeout -> an ISR took too long
  Task watchdog got triggered -> loop() blocked for 5+ seconds

  Turn the backtrace into line numbers:
```

```bash
# with the ESP Exception Decoder plugin, or by hand:
xtensa-esp32-elf-addr2line -pfiaC -e build/sketch.ino.elf 0x400d1234
```

Checking you haven't run out of RAM

```cpp
/*
1. random reboots and corrupted variables usually mean memory, not logic
2. free heap dropping steadily over hours is a leak
3. stack high water mark is how much headroom the task has left
*/

void memReport() {
    Serial.printf("heap: %u  min ever: %u  stack left: %u\n",
        ESP.getFreeHeap(),
        ESP.getMinFreeHeap(),
        uxTaskGetStackHighWaterMark(NULL));
}

// output: heap: 289340  min ever: 271024  stack left: 4520
```

Isolating the fault by removing things

```
  1. comment everything out of loop() except one line
  2. does it still misbehave?
  3. add half of it back
  4. repeat

  Binary search on your own code. Slow, tedious, and it always
  works, which is more than can be said for staring at it.

  Same on the hardware side: unplug every peripheral, add them
  back one at a time. An I2C device holding SDA low takes the
  whole bus down and looks like a software bug.
```

Symptom to cause

|Symptom|Likely cause|
|---|---|
|Board reboots the moment WiFi connects|Brownout, weak USB port or supply|
|Reboot loop with no message|Bad strapping pin, or a short|
|Reboots after minutes or hours|Memory leak, or stack overflow|
|`Guru Meditation LoadProhibited`|Null or dangling pointer|
|`Interrupt wdt timeout`|`Serial.print` or `delay` inside an ISR|
|`Task watchdog got triggered`|A blocking loop, no `delay`/yield|
|Serial prints garbage|Baud mismatch, see [[Serial and UART (ESP32)]]|
|Sensor reads 0 or -1 always|Wiring, address, or missing pull-ups|
|`analogRead` returns 0 with WiFi on|ADC2 pin, move to 32-39|
|Value changes only sometimes|Missing `volatile`, see [[Volatile and interrupt safety]]|
|Works only when you touch it|Floating input, missing pull-up, bad solder joint|
|Button fires 5 times per press|No debounce, see [[State machines in embedded C]]|
|Pin does nothing at all|GPIO 34-39 are input only|
|Works on USB, dead on battery|Not enough current, or brownout|
|Upload fails, dots forever|Hold BOOT, see [[Setting up the toolchain (ESP32)]]|

Toggling a pin to measure timing

```cpp
/*
1. when a print would change the timing you're measuring, toggle a pin instead
2. costs ~20ns, and a logic analyser shows you the exact width
3. this is how you measure ISR duration
*/

void IRAM_ATTR onTimer() {
    GPIO.out_w1ts = (1 << 27);      // pin high on entry

    doTheWork();

    GPIO.out_w1tc = (1 << 27);      // pin low on exit
}

// then look at pin 27 on a scope, see [[Logic analyser and Oscilloscope]]
```

Real debugging, when prints aren't enough

```
  The ESP32 has JTAG built in. With an ESP-Prog adapter (or a
  built-in USB-JTAG on the S3/C3) you get real breakpoints, stepping
  and variable inspection through OpenOCD and gdb:

    idf.py openocd gdb

  Worth setting up if you're doing this seriously. For a first
  project, prints and an LED get you most of the way.
```
