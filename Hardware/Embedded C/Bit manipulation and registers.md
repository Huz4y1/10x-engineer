Hardware is controlled by flipping individual bits inside registers, so bitwise operators stop being trivia and become the everyday tools.

The operators

|Operator|Name|Does|Example|
|---|---|---|---|
|`&`|AND|1 only if both are 1|`0b1100 & 0b1010` = `0b1000`|
|`\|`|OR|1 if either is 1|`0b1100 \| 0b1010` = `0b1110`|
|`^`|XOR|1 if they differ|`0b1100 ^ 0b1010` = `0b0110`|
|`~`|NOT|Flips every bit|`~0b1100` = `0b0011` (in 4 bits)|
|`<<`|Left shift|Moves bits up, fills with 0|`0b0001 << 3` = `0b1000`|
|`>>`|Right shift|Moves bits down|`0b1000 >> 2` = `0b0010`|

The four things you actually do

```c
/*
1. these four lines are 90% of all embedded bit work
2. (1 << n) builds a mask with only bit n set
3. read them as: OR to set, AND-NOT to clear, XOR to toggle, AND to test
*/

#include <stdint.h>

uint8_t reg = 0b00000000;

reg |=  (1 << 3);          // SET bit 3      -> 0b00001000
reg &= ~(1 << 3);          // CLEAR bit 3    -> 0b00000000
reg ^=  (1 << 3);          // TOGGLE bit 3   -> 0b00001000

if (reg & (1 << 3)) {      // TEST bit 3     -> true
    // bit 3 is set
}
```

Why each one works

```
  SET,  reg |= (1 << 3)

    reg    0 0 1 0 0 1 0 1
    mask   0 0 0 0 1 0 0 0     OR: anything OR 0 stays,
    ────── ─────────────────       anything OR 1 becomes 1
    result 0 0 1 0 1 1 0 1
                 ▲ forced to 1, everything else untouched


  CLEAR, reg &= ~(1 << 3)

    mask     0 0 0 0 1 0 0 0
    ~mask    1 1 1 1 0 1 1 1
    reg      0 0 1 0 1 1 0 1     AND: anything AND 1 stays,
    ──────── ───────────────          anything AND 0 becomes 0
    result   0 0 1 0 0 1 0 1
                   ▲ forced to 0


  TOGGLE, reg ^= (1 << 3)

    reg      0 0 1 0 1 1 0 1
    mask     0 0 0 0 1 0 0 0     XOR: 1 flips it, 0 leaves it
    ──────── ───────────────
    result   0 0 1 0 0 1 0 1
                   ▲ flipped
```

Masks, for working with several bits at once

```c
/*
1. a mask is just a pattern of bits you care about
2. name them, never scatter magic numbers through the code
3. an enable-and-configure step usually clears a field then ORs the new value in
*/

#define BIT_ENABLE   (1 << 0)
#define BIT_SLEEP    (1 << 6)

#define MODE_MASK    (0b111 << 2)     // bits 2,3,4 are a 3-bit field
#define MODE_FAST    (0b101 << 2)

uint8_t config = 0;

config |= BIT_ENABLE;                  // one bit

config &= ~MODE_MASK;                  // clear the whole field first
config |=  MODE_FAST;                  // then write the new value in

// checking several at once
if ((config & (BIT_ENABLE | BIT_SLEEP)) == BIT_ENABLE) {
    // enabled AND not sleeping
}
```

Pulling a value back out of a field

```c
/*
1. mask off the bits you want
2. shift them down to position 0
3. now it's an ordinary small integer
*/

uint8_t status = 0b10110100;

uint8_t mode = (status & MODE_MASK) >> 2;

printf("%d\n", mode);        // output: 5
```

Handy tricks

```c
uint8_t x = 0b00101100;

x & 1                        // 1 if odd, 0 if even
x >>= 1;                     // divide by 2, cheaper than /
x <<= 3;                     // multiply by 8
x & (x - 1)                  // clears the lowest set bit
!(x & (x - 1))               // true if x is a power of two

// swapping the halves of a byte
uint8_t swapped = (x << 4) | (x >> 4);

// building a 16-bit value from two bytes, exactly what I2C sensors need
uint8_t  high = 0x1A, low = 0x2B;
uint16_t value = ((uint16_t) high << 8) | low;      // 0x1A2B
```

Direct register writes versus digitalWrite

```cpp
/*
1. digitalWrite is a function call that validates the pin, looks up which
   port it lives on, works out the bit, and then writes it
2. writing the register does only the last step
3. same result, roughly 20-50x faster
*/

// the friendly way
digitalWrite(13, HIGH);

// the ESP32 register way, GPIO 0-31 live in these two registers
GPIO.out_w1ts = (1 << 13);      // Write 1 To Set
GPIO.out_w1tc = (1 << 13);      // Write 1 To Clear

// or the middle ground, still fast, still readable
#include "driver/gpio.h"
gpio_set_level(GPIO_NUM_13, 1);
```

```
  Rough cost of toggling a pin on an ESP32 at 240MHz:

   digitalWrite      ~ 1000 ns   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
   gpio_set_level    ~  100 ns   ▓▓
   GPIO.out_w1ts     ~   20 ns   ▓

  Irrelevant for blinking an LED. Decisive for bit-banging a
  protocol, or inside an interrupt.
```

Set-and-clear registers, and why they exist

```c
/*
1. the obvious way is read-modify-write, which is three operations
2. an interrupt landing in the middle corrupts it, see [[Volatile and interrupt safety]]
3. w1ts/w1tc registers do it in one atomic write, no read needed
*/

// racy
GPIO.out |= (1 << 13);          // read, OR, write back  <- interruptible

// safe and faster
GPIO.out_w1ts = (1 << 13);      // one write, cannot be interrupted halfway
```

The gotcha

```
  1 << 31 on a signed int is undefined behaviour. Use 1UL << 31,
  and 1ULL << n for anything past bit 31 (the ESP32 deep sleep pin
  masks need exactly this).

  Precedence bites constantly. == binds TIGHTER than &:

    if (reg & MASK == 0)      wrong, parsed as reg & (MASK == 0)
    if ((reg & MASK) == 0)    right

  Always bracket bitwise expressions.

  And registers are volatile. Reading one twice may give different
  answers, so the compiler must not cache it.
```
