An ESP32 running flat out eats a battery in hours, asleep it lasts months, and the difference is a few lines of code.

The sleep modes

|Mode|Current|CPU|RAM kept|WiFi|Wake time|
|---|---|---|---|---|---|
|Active, WiFi on|160-260 mA|running|yes|yes|-|
|Active, WiFi off|~40 mA|running|yes|no|-|
|Modem sleep|~20 mA|running|yes|off between beacons|instant|
|Light sleep|~0.8 mA|paused|yes|paused|~1 ms, carries on where it was|
|Deep sleep|~10 uA|off|only RTC memory|off|~300 ms, restarts from setup()|
|Hibernation|~2.5 uA|off|almost nothing|off|restart|

```
  what deep sleep actually does

   ┌─────────────────────────────────────────────┐
   │  ESP32                                      │
   │  ┌──────────────┐   ┌────────────────────┐  │
   │  │ CPU cores    │   │ RTC domain         │  │
   │  │ WiFi / BT    │   │  - RTC timer       │  │
   │  │ main SRAM    │   │  - 8KB RTC memory  │  │
   │  │ most GPIO    │   │  - RTC GPIO pins   │  │
   │  │              │   │  - touch, ULP      │  │
   │  │   POWERED    │   │    STAYS ON        │  │
   │  │     OFF      │   │                    │  │
   │  └──────────────┘   └────────────────────┘  │
   └─────────────────────────────────────────────┘

  Waking is a reboot. setup() runs from the top again.
  Ordinary variables are gone. loop() is never resumed.
```

Timer wake, the common case

```cpp
/*
1. RTC_DATA_ATTR keeps a variable in RTC memory, so it survives deep sleep
2. the timer takes MICROSECONDS, hence the multiply
3. deep_sleep_start() never returns, nothing after it runs
*/

#define uS_TO_S 1000000ULL
#define SLEEP_SECONDS 30

RTC_DATA_ATTR int bootCount = 0;

void setup() {
    Serial.begin(115200);
    delay(100);

    bootCount++;
    Serial.printf("wake number %d\n", bootCount);

    float temp = readSensor();
    Serial.println(temp);

    esp_sleep_enable_timer_wakeup(SLEEP_SECONDS * uS_TO_S);

    Serial.println("sleeping");
    Serial.flush();                  // let the print finish before power cuts

    esp_deep_sleep_start();          // nothing below this line ever runs
}

void loop() {}                       // never reached

// output: wake number 1
//         21.40
//         sleeping
//         (30 seconds of ~10uA)
//         wake number 2
```

Wake sources

|Source|Function|Note|
|---|---|---|
|Timer|`esp_sleep_enable_timer_wakeup(us)`|Most common|
|One pin|`esp_sleep_enable_ext0_wakeup(pin, level)`|RTC pins only|
|Several pins|`esp_sleep_enable_ext1_wakeup(mask, mode)`|A bitmask of RTC pins|
|Touch|`esp_sleep_enable_touchpad_wakeup()`|Touch pads T0-T9|
|ULP coprocessor|`esp_sleep_enable_ulp_wakeup()`|Polls a sensor while asleep|

RTC-capable pins: 0, 2, 4, 12, 13, 14, 15, 25, 26, 27, 32, 33, 34, 35, 36, 39. Anything else cannot wake the chip.

Waking on a button

```cpp
/*
1. ext0 watches one RTC pin for a level
2. 0 means wake when it goes LOW, which suits a button to GND
3. the internal pull-up must be kept alive through sleep or the pin floats and wakes randomly
*/

#define BUTTON GPIO_NUM_33

void setup() {
    Serial.begin(115200);
    delay(100);

    Serial.println("awake");

    esp_sleep_enable_ext0_wakeup(BUTTON, 0);        // wake on LOW

    rtc_gpio_pullup_en(BUTTON);                     // keep the pull-up during sleep
    rtc_gpio_pulldown_dis(BUTTON);

    delay(3000);
    Serial.println("sleeping, press the button");
    esp_deep_sleep_start();
}
```

Waking on touch

```cpp
/*
1. a touch pad is just a wire or a bit of copper on a GPIO
2. touching it lowers the reading, so the threshold is a value BELOW the idle reading
3. print touchRead() while awake to pick a sensible threshold
*/

#define THRESHOLD 40

void onTouch() {}      // must exist, can be empty

void setup() {
    Serial.begin(115200);
    delay(100);

    Serial.println(touchRead(T3));    // output: 72 idle, 18 when touched

    touchAttachInterrupt(T3, onTouch, THRESHOLD);
    esp_sleep_enable_touchpad_wakeup();

    delay(2000);
    esp_deep_sleep_start();
}
```

Finding out why it woke

```cpp
void printWakeReason() {
    esp_sleep_wakeup_cause_t reason = esp_sleep_get_wakeup_cause();

    switch (reason) {
        case ESP_SLEEP_WAKEUP_TIMER:    Serial.println("timer");       break;
        case ESP_SLEEP_WAKEUP_EXT0:     Serial.println("pin");         break;
        case ESP_SLEEP_WAKEUP_EXT1:     Serial.println("pins");        break;
        case ESP_SLEEP_WAKEUP_TOUCHPAD: Serial.println("touch");       break;
        default:                        Serial.println("power on");    break;
    }
}

// output: power on
//         timer
//         pin
```

Light sleep, when you need to carry on where you left off

```cpp
/*
1. light sleep pauses instead of restarting, variables and WiFi state survive
2. costs about 100x more current than deep sleep, but resumes in ~1ms
3. execution continues on the NEXT line, not from setup()
*/

esp_sleep_enable_timer_wakeup(500000);      // 0.5 s
esp_light_sleep_start();

Serial.println("carried on right here");    // this DOES run
```

Battery life, roughly

```
  A 2000mAh 18650, duty cycled: wake 3 seconds, send over WiFi, sleep.

  awake:   3s at 160mA
  asleep: 57s at 10uA

  average = (3*160 + 57*0.01) / 60 ≈ 8.0 mA
  life    = 2000 / 8.0 ≈ 250 hours ≈ 10 days
```

|Behaviour|Average current|2000mAh 18650 lasts|
|---|---|---|
|Always on, WiFi connected|~160 mA|~12 hours|
|Always on, WiFi off|~40 mA|~2 days|
|Wake 3s every minute|~8 mA|~10 days|
|Wake 3s every 10 minutes|~0.8 mA|~3 months|
|Wake 3s every hour|~0.15 mA|~1.5 years (battery self-discharge wins first)|

Getting the current down further

```cpp
/*
1. connecting to WiFi is the expensive part, 2-5 seconds of radio
2. a static IP skips DHCP and saves a second or more
3. turn the radio off explicitly before sleeping
*/

WiFi.config(IPAddress(192,168,1,50),
            IPAddress(192,168,1,1),
            IPAddress(255,255,255,0));

WiFi.begin(ssid, pass);
// ... do the work ...

WiFi.disconnect(true);
WiFi.mode(WIFI_OFF);
esp_deep_sleep_start();

setCpuFrequencyMhz(80);      // and while awake, 80MHz instead of 240 roughly halves it
```

The gotcha

```
  A DevKit board will NEVER reach 10uA. The onboard USB-serial chip
  and the power LED alone draw 5-20mA whatever the ESP32 does. The
  low numbers above need a bare module, or a board designed for
  battery use, or the power LED desoldered.

  Serial.flush() before sleeping, otherwise the last print is cut off
  mid-word and you'll think the board crashed.

  Everything not in RTC_DATA_ATTR is wiped on wake. Counters, WiFi
  credentials you fetched, sensor baselines - all gone.

  Powering from 3V3 while USB is also plugged in can back-feed the
  regulator. Pick one.
```
