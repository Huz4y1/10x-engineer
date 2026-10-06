PWM fakes an analogue output by switching a pin on and off very fast, and changing how much of the time it spends on.

What duty cycle means

```
  duty 25%
       ┌──┐        ┌──┐        ┌──┐
  3V3 ─┘  └────────┘  └────────┘  └────
       ├──┤
       on   off

  duty 50%
       ┌────┐    ┌────┐    ┌────┐
  3V3 ─┘    └────┘    └────┘    └────

  duty 90%
       ┌────────┐┌────────┐┌────────┐
  3V3 ─┘        └┘        └┘        └─

  the pin is always fully on or fully off, the LED just
  can't switch that fast so your eye averages it
```

The ESP32 does not use analogWrite, it uses LEDC

```cpp
/*
1. LEDC is the ESP32's PWM peripheral, 16 independent channels
2. you set up a channel (frequency + resolution), then attach a pin to it
3. then write a duty value to the channel
*/

#define LED 13

#define CHANNEL    0
#define FREQ       5000     // Hz
#define RESOLUTION 8        // bits, so duty is 0-255

void setup() {
    ledcSetup(CHANNEL, FREQ, RESOLUTION);
    ledcAttachPin(LED, CHANNEL);
}

void loop() {
    ledcWrite(CHANNEL, 0);     // off
    delay(500);
    ledcWrite(CHANNEL, 128);   // half brightness
    delay(500);
    ledcWrite(CHANNEL, 255);   // full
    delay(500);
}
```

Resolution decides the maximum duty number

|Resolution|Duty range|
|---|---|
|8 bit|0 - 255|
|10 bit|0 - 1023|
|12 bit|0 - 4095|
|16 bit|0 - 65535|

Frequency and resolution trade against each other

```
  The peripheral runs from an 80 MHz clock, so:

      max frequency = 80,000,000 / 2^resolution

   8-bit  ->  up to 312 kHz
  10-bit  ->  up to  78 kHz
  16-bit  ->  up to 1.2 kHz

  Ask for a combination that doesn't fit and ledcSetup returns 0
  and the pin does nothing.
```

Fading an LED

```cpp
/*
1. walk the duty value up then back down
2. 5 kHz is well above what the eye can see, so no flicker
3. brightness looks non-linear because eyes are logarithmic, that's expected
*/

void loop() {
    for (int duty = 0; duty <= 255; duty++) {
        ledcWrite(CHANNEL, duty);
        delay(5);
    }
    for (int duty = 255; duty >= 0; duty--) {
        ledcWrite(CHANNEL, duty);
        delay(5);
    }
}
```

Driving a motor's speed

```cpp
/*
1. a GPIO can supply about 12mA, a motor wants hundreds, so a driver chip sits between
2. PWM the driver's enable/speed input, not the motor directly
3. motors like a lower frequency, 1-20 kHz. Under 20 kHz it whines audibly
*/

#define MOTOR_PWM 14
#define MOTOR_DIR 27

#define M_CHANNEL 1

void setup() {
    ledcSetup(M_CHANNEL, 20000, 8);      // 20 kHz, above hearing
    ledcAttachPin(MOTOR_PWM, M_CHANNEL);
    pinMode(MOTOR_DIR, OUTPUT);
}

void loop() {
    digitalWrite(MOTOR_DIR, HIGH);
    ledcWrite(M_CHANNEL, 80);            // ~31% speed, may not be enough to start it
    delay(2000);

    ledcWrite(M_CHANNEL, 255);           // full
    delay(2000);

    ledcWrite(M_CHANNEL, 0);             // stop
    delay(2000);
}
```

```
  never wire a motor straight to a pin

   GPIO ──X── motor          burns the pin, and the back-EMF
                             spike can kill the chip

   GPIO ──> [driver: L298N / DRV8833 / a MOSFET + flyback diode] ──> motor
                        ▲
                  separate power supply, common GND with the ESP32
```

A servo, which is just PWM with a fussy pulse width

```cpp
/*
1. hobby servos want a 50 Hz signal, pulse 1ms = 0 deg, 2ms = 180 deg
2. at 50 Hz one period is 20ms, so 1ms is 5% duty and 2ms is 10%
3. 16-bit resolution gives fine enough steps
*/

#define SERVO 26
#define S_CH  2

void setup() {
    ledcSetup(S_CH, 50, 16);          // 0-65535 duty
    ledcAttachPin(SERVO, S_CH);
}

void writeAngle(int deg) {
    // 1ms..2ms of a 20ms period  ->  3277..6554 of 65535
    int duty = map(deg, 0, 180, 3277, 6554);
    ledcWrite(S_CH, duty);
}

void loop() {
    writeAngle(0);    delay(1000);
    writeAngle(90);   delay(1000);
    writeAngle(180);  delay(1000);
}
```

The ESP-IDF way, more verbose but the same idea

```c
// ESP-IDF splits it into a timer and a channel, which is what LEDC really is

#include "driver/ledc.h"

ledc_timer_config_t timer = {
    .speed_mode      = LEDC_LOW_SPEED_MODE,
    .duty_resolution = LEDC_TIMER_8_BIT,
    .timer_num       = LEDC_TIMER_0,
    .freq_hz         = 5000,
    .clk_cfg         = LEDC_AUTO_CLK,
};
ledc_timer_config(&timer);

ledc_channel_config_t ch = {
    .gpio_num   = 13,
    .speed_mode = LEDC_LOW_SPEED_MODE,
    .channel    = LEDC_CHANNEL_0,
    .timer_sel  = LEDC_TIMER_0,
    .duty       = 128,
    .hpoint     = 0,
};
ledc_channel_config(&ch);
```

The gotcha

```
  analogWrite() does now exist in newer ESP32 Arduino cores, but it
  quietly grabs an LEDC channel for you. Mixing it with ledcSetup on
  the same pin fights over the channel.

  Also there are only 16 channels but only 8 independent timers,
  channels 0/1 share a timer, 2/3 share, and so on. Two channels on
  the same timer must run at the same frequency.

  PWM is not a real voltage. Feeding it into something that expects
  analogue needs an RC filter, or use the DAC on GPIO 25/26.
```
