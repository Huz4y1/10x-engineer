A multimeter tells you the level, these two tell you what the signal is doing over time, which is the difference between guessing and knowing.

What each one is for

|                |Logic analyser|Oscilloscope|
|---|---|---|
|Shows|Is it 1 or 0, over time|The actual voltage shape|
|Channels|8, 16, 32 at once|Usually 2 or 4|
|Decodes protocols|Yes, I2C SPI UART CAN and more|Only on better models|
|Sees noise, ringing, sag|No|Yes|
|Sees analogue signals|No|Yes|
|Price to get started|£10 for a clone Saleae|£150+ for a usable one|
|Software|Free (sigrok / PulseView, Saleae Logic)|On the instrument|

```
  the SAME signal, seen by each

  logic analyser                oscilloscope

   ┌───┐   ┌───┐                  ┌╮__┐╮   ┌╮__┐╮
  ─┘   └───┘   └──               ─┘     ╰──┘     ╰──
                                  ▲
   clean 1s and 0s                overshoot, ringing,
   perfect for reading            a slow rising edge,
   protocol data                  a supply sagging

  Debugging "the sensor returns nothing"  -> logic analyser
  Debugging "it works sometimes"          -> oscilloscope
```

When you actually need one

|Problem|Tool|
|---|---|
|"Is my code even sending anything?"|Logic analyser|
|"Is the sensor replying?"|Logic analyser|
|"Wrong I2C address, or wrong register?"|Logic analyser|
|"Is my PWM the frequency I asked for?"|Either|
|"How long does my ISR actually take?"|Either, toggle a pin|
|"Is my SPI mode right?"|Logic analyser (shows the clock edges)|
|"Why does it reboot when the motor starts?"|Oscilloscope (supply dip)|
|"My sensor reading is noisy"|Oscilloscope|
|"Is this pin high?"|Multimeter, don't overthink it|

Hooking a logic analyser up

```
   ESP32                 analyser              PC
  ┌────────┐            ┌─────────┐
  │ SDA 21 ├────────────┤ CH0     │
  │ SCL 22 ├────────────┤ CH1     ├──USB──▶ PulseView
  │ GND    ├────────────┤ GND     │
  └────────┘            └─────────┘
                             ▲
                    GND is not optional, it's the
                    reference every measurement is against

  It's passive, it just listens. Clip onto the running
  circuit, nothing to configure on the ESP32 side.
```

Reading an I2C capture

```
  SDA ──┐ ┌─┐ ┌───┐ ┌─┐   ┌───────┐ ┌─┐ ┌─────
        └─┘ └─┘   └─┘ └───┘       └─┘ └─┘
  SCL ────┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌─────
          └─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘
        ▲                            ▲    ▲
      START                         ACK  STOP
   SDA falls while                SDA pulled low
   SCL is still high              by the SLAVE

  The decoder in PulseView turns that into:

    START | 0x76 W | ACK | 0xF7 | ACK | START | 0x76 R | ACK | 0x51 | NAK | STOP

  Which immediately tells you:
    - the address you're actually sending (vs the one you meant)
    - whether the device ACKed (it exists) or NAKed (it doesn't)
    - the register and the data
```

What each I2C failure looks like on screen

|What you see|Means|
|---|---|
|Nothing at all, both lines flat high|Your code isn't sending. Wrong pins, or `Wire.begin` missing|
|Both lines flat LOW|No pull-ups, or a device holding the bus|
|Address then NAK|Wrong address, or the device isn't powered|
|Address ACK then garbage|Right device, wrong register or wrong byte order|
|Edges look like slow curves|Pull-ups too weak, raise to 2.2k|

Reading SPI

```
  CS   ──┐                                    ┌──
         └────────────────────────────────────┘
  SCK  ────┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌──────────
           └┘└┘└┘└┘└┘└┘└┘└┘
  MOSI ──┬──────┬───────┬────────
         │ 0x8F │  0x00 │
  MISO ──┬──────┬───────┬────────
         │ 0x00 │  0x6B │

  CS low frames the whole transaction. 8 clock pulses = 1 byte.
  MOSI is what you sent, MISO is what came back.

  If MISO is flat high the whole time, the device isn't
  responding: check CS, power, and the SPI mode.
```

Oscilloscope basics, the three knobs that matter

|Control|Does|
|---|---|
|Volts/div|Vertical zoom. 1V/div is right for 3.3V logic|
|Time/div|Horizontal zoom. Start wide, zoom in|
|Trigger|The single most important one. Sets when it starts capturing|

```
  Without a trigger the waveform slides across the screen
  unreadably. Set it to "rising edge at 1.6V" and every capture
  starts at the same point, so the trace stands still.

  Single-shot trigger is how you catch a one-off event, like a
  glitch that only happens at startup.
```

What a scope shows that a logic analyser can't

```
  supply dipping when a motor starts

   3.3V ──────╮                    ╭──────────
              │                    │
   2.6V       ╰────────────────────╯
              ▲
        motor starts, supply sags below the ESP32's
        brownout threshold, board reboots

  A logic analyser sees a clean 3.3V then nothing. The scope
  shows you exactly why.


  ringing on a fast edge

        ╭╮╭╮__________
   ─────╯╰╯╰

  Long unshielded jumper wires on a fast SPI clock. This is why
  a 20MHz SPI works on a PCB and fails on a breadboard.
```

Cheap and good enough

```
  £10-15  8-channel USB logic analyser ("Saleae clone", CY7C68013A)
          + PulseView (free, open source, decodes 100+ protocols)
          Handles up to 24 MHz. Ample for I2C, SPI, UART.
          This is the single best value tool in electronics.

  £60     DSO138 style kit scope. Slow, one channel. Better than
          nothing for seeing a supply sag, not much else.

  £150+   A real 2-channel 100MHz digital scope. Buy this when
          you find yourself repeatedly guessing about analogue
          behaviour, not before.

  Buy the logic analyser first. For microcontroller work it
  solves far more problems per pound than a scope does.
```

Using an ESP32 pin as a poor man's probe

```cpp
/*
1. no analyser to hand, but you want to time something
2. toggle a spare pin around the code you care about
3. then measure the pulse width with anything, even another ESP32
4. costs ~20ns, unlike a Serial.print which costs ~100us and
   changes the timing you were trying to measure
*/

#define PROBE 27

void IRAM_ATTR onTimer() {
    GPIO.out_w1ts = (1 << PROBE);      // high on entry

    doWork();

    GPIO.out_w1tc = (1 << PROBE);      // low on exit
}

// the width of that pulse IS your ISR duration
// see [[Debugging embedded code]]
```

The gotcha

```
  Ground first. Every measurement is relative to ground, and a
  probe without a ground connection shows you plausible-looking
  nonsense.

  A 10x probe on a scope divides the signal by ten. If the
  software isn't told, every voltage you read is wrong by 10x.

  Sample rate has to be well above the signal. Capturing a
  1 MHz SPI clock at 2 MHz gives you aliased garbage. Rule of
  thumb: at least 4x, ideally 10x the fastest edge.

  And a scope probe on a mains circuit, with the ground clip
  connected to anything live, shorts through the mains earth.
  Don't put a normal scope on mains.
```

---

**Where it's used:** [[20 — EMBEDDED]] - decoding [[I2C (ESP32)]] and [[SPI (ESP32)]] when a sensor stays silent.
