Fixed point means storing 21.47 as the integer 2147 and remembering the decimal point yourself, which is much faster on a chip with no floating point unit.

Why floats hurt

|Chip|Float support|Cost of a float multiply|
|---|---|---|
|Arduino Uno (AVR)|None, all software|~100x an int multiply|
|ESP32|Hardware FPU, `float` only|About the same as an int|
|ESP32, `double`|Software|~50x slower than `float`|
|ESP32-C3|None at all|Software, very slow|

```
  So the honest answer on an ESP32:

    float   is fine, there's an FPU, use it
    double  is not, it's emulated. And 3.14 is a double literal,
            3.14f is a float. That trailing f matters.

  On anything smaller, or inside an interrupt, or on a battery,
  integers still win outright.
```

The trap even on an ESP32

```cpp
/*
1. 3.3 with no suffix is a DOUBLE, so this promotes everything to double
2. the FPU can't help, it gets emulated in software
3. adding one character makes it ~50x faster
*/

float volts = raw * (3.3 / 4095.0);      // slow, double maths
float volts = raw * (3.3f / 4095.0f);    // fast, float maths
```

Fixed point, the idea

```
  Pick a scale factor and stick to it.

  scale 100:   21.47  is stored as  2147
               0.05   is stored as     5
               100.0  is stored as 10000

  The integer 2147 means "2147 hundredths".
  Nothing in the hardware knows this. It's a convention you keep.

   2 1 4 7
   ─────────
   2 1 . 4 7      the point is imaginary, it lives in your head
```

Adding and subtracting just work

```c
/*
1. same scale in, same scale out
2. no adjustment needed at all
*/

int32_t a = 2147;      // 21.47
int32_t b =  350;      //  3.50

int32_t sum = a + b;   // 2497 = 24.97   correct
```

Multiplying and dividing need a correction

```c
/*
1. multiplying two scaled numbers scales the result TWICE
2. so divide by the scale once to bring it back
3. dividing loses the scale entirely, so multiply up first
4. use a wider type in the middle or it overflows
*/

#define SCALE 100

int32_t a = 2147;      // 21.47
int32_t b =  350;      //  3.50

int32_t product = ((int64_t) a * b) / SCALE;    // 7514 = 75.14
int32_t quotient = ((int64_t) a * SCALE) / b;   //  613 =  6.13
```

Printing it

```c
/*
1. integer divide for the whole part, modulo for the fraction
2. %02d pads so 21.05 doesn't print as 21.5
*/

int32_t temp = 2147;

printf("%d.%02d\n", temp / 100, temp % 100);

// output: 21.47

// negatives need care, -2147 / 100 is -21 and -2147 % 100 is -47
int32_t t = -2147;
printf("%s%d.%02d\n", t < 0 ? "-" : "", abs(t) / 100, abs(t) % 100);

// output: -21.47
```

A worked example, ADC to millivolts

```cpp
/*
1. the float version does a divide and a multiply in floating point
2. the integer version multiplies first so nothing is lost to truncation
3. result is in millivolts, an int, exact enough for anything real
*/

// float version
float volts = raw * (3.3f / 4095.0f);
Serial.println(volts, 3);              // output: 1.650

// fixed point version, millivolts
uint32_t mv = ((uint32_t) raw * 3300) / 4095;
Serial.printf("%lu.%03lu\n", mv / 1000, mv % 1000);   // output: 1.650

// raw = 2048:  2048 * 3300 = 6758400, / 4095 = 1650 mV
```

Order matters enormously

```c
/*
1. integer division truncates, so dividing first throws the value away
2. multiply first, divide last, always
*/

uint32_t raw = 2048;

uint32_t bad  = (raw / 4095) * 3300;    // 0 * 3300  = 0      wrong
uint32_t good = (raw * 3300) / 4095;    // 6758400 / 4095 = 1650   right
```

Powers of two are the fastest scale

```c
/*
1. scaling by 256 turns the correction divide into a shift
2. a shift is one cycle, a divide can be 20+
3. this is called Q8.8: 8 integer bits, 8 fractional bits
*/

#define Q 8                       // scale = 256

int32_t a = 21.47 * 256;          // 5496, computed at compile time
int32_t b =  3.50 * 256;          //  896

int32_t product = (a * b) >> Q;   // shift instead of / 256
int32_t whole   = a >> Q;         // 21
```

|Format|Scale|Range in 16 bits|Smallest step|
|---|---|---|---|
|Q8.8|256|-128 to 127.996|0.0039|
|Q12.4|16|-2048 to 2047.9|0.0625|
|scale 100|100|-327 to 327|0.01|
|scale 1000|1000|-32 to 32|0.001|

Picking a scale

```
  Ask two questions:

  1. how precise does it need to be?
     temperature to 0.1 C   -> scale 10
     volts to 1 mV          -> scale 1000

  2. what's the biggest value?
     scale 1000 in an int32_t tops out at 2,147,483 units
     which is plenty. In an int16_t it's 32 units. It isn't.

  When in doubt: int32_t, scale 1000, and check the overflow.
```

Avoiding maths entirely

```c
/*
1. a lookup table replaces a calculation with an array read
2. computed once, at compile time, costs flash instead of cycles
3. classic for sin/cos, thermistor curves, gamma correction
*/

// sine, scaled by 1000, one entry per degree for a quarter turn
static const int16_t sinTable[91] = {
       0,   17,   35,   52,   70,   87,  105,  122,  139,  157,
     174,  /* ... */  985,  990,  995,  998, 1000
};

int16_t sinDeg(int deg)
{
    deg %= 360;
    if (deg <=  90) return  sinTable[deg];
    if (deg <= 180) return  sinTable[180 - deg];
    if (deg <= 270) return -sinTable[deg - 180];
    return                 -sinTable[360 - deg];
}

// sinDeg(30)  ->  500   meaning 0.500
```

The gotcha

```
  Overflow is silent. int16_t a = 200 * 200 gives you 24 in a
  wrapped-around value, no warning, no crash, just a wrong answer.
  Cast to int32_t or int64_t before multiplying.

  Never compare floats with ==, they don't land exactly:
      0.1f + 0.2f == 0.3f     is false
  Compare a difference against a tolerance, or use fixed point
  where == does exactly what you'd expect.

  And don't do this prematurely. On an ESP32 with an FPU, floats
  are fine for reading a sensor once a second. Reach for fixed
  point when you're in an ISR, on a battery, or on a smaller chip.
```
