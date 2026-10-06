It's the same language with the same rules, but everything you took for granted from the operating system is simply not there.

What's missing

|On a PC|On a microcontroller|
|---|---|
|An OS underneath you|Nothing. Your code IS the system|
|`printf` to a terminal|A UART, if you wired one up|
|`main()` returns and you're done|Returning from main is a crash or a reboot|
|Gigabytes of RAM|520 KB on an ESP32, 2 KB on an Uno|
|`malloc` whenever you like|Technically works, in practice avoided|
|A filesystem|Maybe a flash partition, if you set it up|
|A debugger, breakpoints|Serial prints and a blinking LED|
|Crash = one process dies|Crash = the whole board reboots|

The infinite loop, which is the shape of every embedded program

```c
/*
1. there is nothing to return TO, so main never ends
2. for(;;) and while(1) are identical, both are just "forever"
3. if you ever fall off the end of main the chip resets or wanders into garbage
*/

int main(void)
{
    setupHardware();          // runs once

    for (;;)                  // runs until the power goes off
    {
        readSensors();
        updateOutputs();
    }

    return 0;                 // unreachable, and that is deliberate
}
```

Arduino just splits that in half for you

```cpp
/*
1. setup() is the "runs once" part
2. loop() is the body of the infinite loop
3. the framework literally writes the main() around them
*/

void setup() {
    pinMode(13, OUTPUT);
}

void loop() {
    digitalWrite(13, HIGH);
}

// what the Arduino core actually compiles:
//
// int main(void) {
//     initArduino();
//     setup();
//     for (;;) {
//         loop();
//         if (serialEventRun) serialEventRun();
//     }
// }
```

No printf by default

```c
/*
1. printf needs somewhere to print TO, and there's no console
2. so you point it at a UART yourself, or use the framework's Serial
3. and it's expensive: printf pulls in float formatting and can eat 2KB of flash
*/

// bare metal, you write this yourself
int _write(int fd, char *buf, int len)
{
    for (int i = 0; i < len; i++) {
        uart_send_byte(buf[i]);
    }
    return len;
}

// Arduino does it for you
Serial.printf("temp = %d\n", temp);
```

No malloc, in practice

```c
/*
1. malloc still exists and still works
2. but with 520KB and no MMU, repeated malloc/free fragments the heap
3. a device that must run for months cannot afford a slow leak
4. so you allocate everything up front, at a fixed size
*/

// avoided
char *buf = malloc(len);          // how big is len? what if it fails at 3am?

// preferred
static char buf[64];              // exists for the whole program, known at compile time

// and a fixed-size ring buffer instead of a growing list
#define MAX_READINGS 32

static int   readings[MAX_READINGS];
static uint8_t head = 0;

void addReading(int v)
{
    readings[head] = v;
    head = (head + 1) % MAX_READINGS;    // wraps, never grows
}
```

Where your memory actually goes

```
   flash (4 MB)                 RAM (520 KB)
  ┌────────────────┐           ┌────────────────┐
  │ bootloader     │           │ .data          │ globals with a value
  ├────────────────┤           ├────────────────┤
  │ your program   │  copied   │ .bss           │ globals set to 0
  │ (.text)        │ ────────▶ ├────────────────┤
  ├────────────────┤   at      │ heap           │ malloc, grows up
  │ constants      │  boot     │      ↓         │
  │ (.rodata)      │           │      ↑         │
  ├────────────────┤           │ stack          │ locals, grows down
  │ initial values │           └────────────────┘
  │ for .data      │
  └────────────────┘            they meet = stack overflow = reboot

  A big local array is on the STACK, which is small. A big global
  or static array is in .bss, which is the whole of RAM.
```

Types you should actually use

```c
/*
1. "int" is 16 bits on an AVR and 32 on an ESP32, so it means nothing portable
2. stdint.h types say exactly what you get
3. on 8-bit parts, using uint8_t instead of int genuinely makes code smaller and faster
*/

#include <stdint.h>

uint8_t  count;      // 0 to 255,           1 byte
int8_t   offset;     // -128 to 127,        1 byte
uint16_t adc;        // 0 to 65535,         2 bytes
int32_t  ticks;      // ±2 billion,         4 bytes
uint32_t millis;     // 0 to 4.29 billion,  4 bytes
```

Things that are the same

```c
// pointers, structs, arrays, the preprocessor, all unchanged
// see [[Pointers (C)]] and [[Memory (C)]]

typedef struct {
    uint8_t  pin;
    uint16_t lastValue;
    bool     enabled;
} Sensor;

static Sensor sensors[4] = {
    { .pin = 32, .lastValue = 0, .enabled = true },
    { .pin = 33, .lastValue = 0, .enabled = true },
};
```

The mindset shift

```
  On a PC:      make it work, then make it fast if anyone complains
  On an MCU:    it must fit, it must never stop, and it must never
                block, because there is no scheduler to save you

  Which leads directly to:
    - no delay() in real code      -> [[Non-blocking timing]]
    - no growing data structures   -> fixed arrays
    - no floats where you can help it -> [[Fixed point and avoiding floats]]
    - no printf debugging          -> [[Debugging embedded code]]
```
