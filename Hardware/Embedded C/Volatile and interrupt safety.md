The compiler assumes it is the only thing changing your variables, and on a microcontroller that assumption is wrong.

The bug

```c
/*
1. the compiler reads flag once, sees the loop body never changes it,
   and concludes the condition is constant
2. so it hoists the read out and generates an infinite loop
3. the interrupt does set flag, but the code never looks again
4. this compiles clean, runs fine at -O0, and hangs at -O2
*/

bool flag = false;

void ISR(void)
{
    flag = true;
}

int main(void)
{
    while (!flag) {
        // wait for the interrupt
    }

    doSomething();      // never reached
}
```

What the optimiser actually did

```
  what you wrote                what it compiled to

  loop:                         load  r1, [flag]      once, before the loop
    load  r1, [flag]            loop:
    test  r1                      test r1
    jz    loop                    jz   loop           r1 never changes
                                                      = infinite loop
```

The fix

```c
/*
1. volatile means "this can change without you seeing it happen"
2. it forces a real memory read every single time
3. it costs a few cycles and buys correctness
*/

volatile bool flag = false;
```

When you need it

|Situation|Volatile?|
|---|---|
|Written by an ISR, read by main|Yes|
|Written by main, read by an ISR|Yes|
|A hardware register|Yes, always|
|Shared with another FreeRTOS task|Yes, and a mutex or queue too|
|An ordinary local variable|No|
|A loop counter|No|

What volatile does NOT do

```
  volatile guarantees:  the read/write really happens, in order

  volatile does NOT guarantee:  the operation is atomic

  This is the single most common misunderstanding.
```

The atomicity problem

```c
/*
1. counter++ is not one instruction, it's load, add, store
2. an interrupt can land between any two of them
3. so the ISR's increment is silently thrown away
*/

volatile uint32_t counter = 0;

void loop(void)
{
    counter++;
}

// main:  load  r1, [counter]     r1 = 5
//            ← INTERRUPT, ISR sets counter to 6
//        add   r1, 1             r1 = 6
//        store [counter], r1     counter = 6, the ISR's work is gone
```

Reading something bigger than one word

```c
/*
1. the ESP32 is 32-bit, so a uint32_t read is one instruction and safe
2. a uint64_t, a float pair, or a struct takes several
3. an interrupt in between gives you half old data and half new: a torn read
*/

volatile uint64_t microseconds = 0;   // updated by a timer ISR

void loop(void)
{
    uint64_t t = microseconds;   // TWO loads. Can tear.

    // low word from before the ISR, high word from after
    // = a timestamp that never existed
}
```

The fix, a critical section

```c
/*
1. turn interrupts off, copy, turn them back on
2. keep it as short as physically possible, interrupts are blocked meanwhile
3. copy out first, then do the work outside the critical section
*/

void loop(void)
{
    noInterrupts();
    uint64_t t = microseconds;     // nothing can interrupt this read
    interrupts();

    process(t);                    // slow work happens with interrupts back on
}
```

The pattern that covers most cases

```cpp
/*
1. the ISR does the least possible: set a flag, stash a value
2. main checks the flag, clears it, then does the real work
3. clear the flag BEFORE processing, so an event arriving during
   processing isn't lost
*/

#define BUTTON 4

volatile bool     pressed  = false;
volatile uint32_t pressTime = 0;

void IRAM_ATTR onPress() {
    pressed   = true;
    pressTime = millis();          // 32-bit, single write, safe
}

void setup() {
    Serial.begin(115200);
    pinMode(BUTTON, INPUT_PULLUP);
    attachInterrupt(BUTTON, onPress, FALLING);
}

void loop() {
    if (pressed) {
        pressed = false;           // clear FIRST

        Serial.println(pressTime); // Serial is fine here, never in the ISR
    }
}

// output: 4210
//         6835
```

A flag is safe, a counter is not

```c
volatile bool     flag  = false;    // one write of one value, safe
volatile uint32_t count = 0;        // read-modify-write, NOT safe

// safe way to read and reset a counter
noInterrupts();
uint32_t n = count;
count = 0;
interrupts();
```

The FreeRTOS way, which the ESP32 gives you for free

```cpp
/*
1. a queue is properly thread-safe and works across both cores
2. the ISR pushes, a task pops, no volatile juggling and no torn reads
3. this is what you should reach for once things get bigger than a flag
*/

QueueHandle_t queue;

void IRAM_ATTR onPress() {
    uint32_t t = millis();
    xQueueSendFromISR(queue, &t, NULL);     // note: the FromISR variant
}

void setup() {
    Serial.begin(115200);
    queue = xQueueCreate(10, sizeof(uint32_t));
    attachInterrupt(4, onPress, FALLING);
}

void loop() {
    uint32_t t;
    if (xQueueReceive(queue, &t, portMAX_DELAY)) {
        Serial.println(t);
    }
}
```

The gotcha

```
  volatile on a POINTER is ambiguous, read it right to left:

    volatile int *p;      the int is volatile, the pointer isn't
    int *volatile p;      the pointer is volatile, the int isn't
    volatile int *volatile p;   both

  For a hardware register you almost always want the first.

  And volatile is not a synchronisation primitive. It is only a
  promise to the compiler about reads and writes. Anything more
  than a single-word flag needs interrupts off, or a queue.
```
