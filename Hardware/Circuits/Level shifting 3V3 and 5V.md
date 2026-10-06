The ESP32 runs at 3.3V and its pins are not 5V tolerant, so connecting a 5V output to a GPIO can damage it, but the reverse direction usually works fine for free.

Getting the direction right is most of this note.

The asymmetry

|Direction|Safe?|Why|
|---|---|---|
|ESP32 output 3.3V → 5V device input|**Usually yes**|Most 5V logic reads anything above ~2.0V as HIGH|
|5V device output → ESP32 input|**No**|5V exceeds the pin's maximum and damages it|

That asymmetry is why some connections need no components at all and others need a divider.

The reason 3.3V into 5V works: a 5V CMOS chip's HIGH threshold is typically 0.7 × VCC = 3.5V, which 3.3V does not quite reach. But most real devices, especially anything TTL-compatible, treat 2.0V as HIGH with margin to spare. Servos, ESCs, most sensor TRIG pins, most 5V microcontroller inputs, all read 3.3V happily.

It is not guaranteed. If a 5V input is not responding to your 3.3V signal, that is a genuine cause and you need to shift upwards. But try it first, it usually works.

The reason 5V into 3.3V is not safe: the ESP32's GPIO has protection diodes to its 3.3V rail. A 5V input forward-biases them and dumps current into the supply. Sometimes the pin fails immediately. More often it degrades over weeks and then dies with no obvious cause, which is much harder to diagnose.

When you get away with it

|Connection|Verdict|
|---|---|
|ESP32 GPIO → servo signal wire|Fine, no shifting|
|ESP32 GPIO → ESC signal wire|Fine|
|ESP32 GPIO → HC-SR04 TRIG|Fine|
|ESP32 GPIO → relay module IN|Usually, check it triggers reliably|
|ESP32 GPIO → L298N / DRV8833 inputs|Fine|
|ESP32 GPIO → 5V LED strip data (WS2812)|**Marginal**, often works, sometimes does not|
|HC-SR04 ECHO → ESP32 GPIO|**Not safe**, needs shifting|
|5V Arduino TX → ESP32 RX|**Not safe**|
|5V sensor analogue out → ESP32 ADC|**Not safe**|
|5V I2C device → ESP32 SDA/SCL|**Depends**, see below|

The WS2812 case is a known grey area. The strip's datasheet wants 0.7 × 5V = 3.5V for a HIGH, and 3.3V is below that. Many strips work anyway. Some work at room temperature and fail when warm. If a strip is glitching, that is why, and the fix is a proper shifter or a single 74HCT125 buffer.

Method 1: voltage divider, 5V down to 3.3V

The cheapest fix. Two resistors, works for slow signals.

```
   5V signal ───[1kΩ]───┬───[2kΩ]─── GND
                        │
                  ESP32 GPIO18
```

|From|To|Note|
|---|---|---|
|5V device output|1kΩ resistor leg 1|-|
|1kΩ resistor leg 2|ESP32 GPIO18 and 2kΩ leg 1|junction sits at 3.33V|
|2kΩ resistor leg 2|ESP32 GND|-|
|5V device GND|ESP32 GND|**shared ground, mandatory**|

```
Vout = 5 × 2000 / (1000 + 2000)
     = 5 × 0.667
     = 3.33V
```

Right on the limit. Safer pairs:

|R1|R2|Output from 5V|
|---|---|---|
|1kΩ|2kΩ|3.33V, right at the edge|
|2.2kΩ|3.3kΩ|3.00V, comfortable|
|10kΩ|15kΩ|3.00V, less current but slower|
|1.8kΩ|3.3kΩ|3.24V|

Use 2.2kΩ and 3.3kΩ if you have them. See [[Voltage dividers]].

Limits of the divider: it only shifts **downwards**, it only works one way, and it slows the signal down. The resistance plus the pin's capacitance forms an RC filter that rounds the edges off. Fine up to roughly 100kHz, so fine for an ultrasonic echo pulse or a slow serial link, useless for SPI at 8MHz.

Method 2: MOSFET bidirectional shifter, for I2C

I2C is a problem for the divider because both sides drive the line, so you need something that works in both directions. The standard solution is one small N-channel MOSFET per line, usually a BSS138.

```
                    3V3 ────┬────           5V ────┬────
                            │                      │
                        [10kΩ]                 [10kΩ]
                            │                      │
   ESP32 SDA ───────────────┤ S     D ├──────────── 5V device SDA
                              │  G  │
                              └──┬──┘
                                 │
                            3V3 ─┘
```

|From|To|Note|
|---|---|---|
|ESP32 SDA (GPIO21)|MOSFET source|low voltage side|
|MOSFET drain|5V device SDA|high voltage side|
|MOSFET gate|ESP32 3V3|the low side supply|
|ESP32 3V3|10kΩ to MOSFET source|pull-up on the 3.3V side|
|5V rail|10kΩ to MOSFET drain|pull-up on the 5V side|
|Repeat all of the above for SCL|-|two MOSFETs, four resistors|
|5V device GND|ESP32 GND|shared|

How it works, briefly. When both sides are idle the pull-ups hold them high and the MOSFET is off. If the 3.3V side pulls low, Vgs becomes 3.3V and the MOSFET turns on, pulling the 5V side low too. If the 5V side pulls low, current flows through the MOSFET's internal body diode, dragging the 3.3V side down and then fully switching the MOSFET on. Both directions covered.

You do not need to build this. Buy a **4-channel BSS138 level shifter module** for about a pound. It has four of these on one board, labelled LV, HV, LV1-4, HV1-4 and grounds. Wire LV to 3V3, HV to 5V, both grounds, and pass your signals through.

|Module pin|Connect to|
|---|---|
|LV|ESP32 3V3|
|HV|5V supply|
|GND (both)|Common ground|
|LV1|ESP32 GPIO21 (SDA)|
|HV1|5V device SDA|
|LV2|ESP32 GPIO22 (SCL)|
|HV2|5V device SCL|

Note that many "5V" I2C sensors actually have 3.3V logic internally with an onboard regulator, and their SDA/SCL pull-ups go to 3.3V. Check the module before assuming you need a shifter. Run the I2C scanner sketch and see.

Method 3: dedicated buffer IC

For fast signals or shifting **upwards**, use a proper chip.

|Chip|Direction|Speed|Use for|
|---|---|---|---|
|74HCT125|3.3V → 5V|Fast|WS2812 data, driving 5V logic|
|74AHCT125|3.3V → 5V|Very fast|Same, better|
|TXS0108E|Bidirectional, 8 channel|Fast|General purpose|
|TXB0104|Bidirectional, 4 channel|Fast|Not for I2C, it fights the pull-ups|
|BSS138 module|Bidirectional, 4 channel|Slow-ish|I2C, slow signals|

The `HCT` part of 74HCT125 matters. `HCT` chips have TTL input thresholds, meaning they read anything above 2.0V as HIGH, so a 3.3V input is comfortably recognised while the output swings to a full 5V. A plain `HC` version has CMOS thresholds and would need 3.5V. HC and HCT look identical and are not interchangeable here.

Method 4: just buy the 3.3V version

Often the cheapest, fastest and most reliable fix is to not have the problem.

|5V part|3.3V equivalent|
|---|---|
|HC-SR04|**HC-SR04P** or RCWL-1601, same price|
|5V relay module|3.3V trigger relay module|
|5V WS2812 strip|Cannot avoid, use a 74HCT125|
|5V Arduino talking serial|Use another ESP32|

The HC-SR04P is genuinely the answer for the most common case. It costs the same, works down to 3V, and saves you two resistors and the risk.

The worked case: HC-SR04 on an ESP32

The standard HC-SR04 needs 5V to transmit properly, so you power it from 5V but must protect the ECHO line.

```
   5V ──────────────────── HC-SR04 VCC

   ESP32 GPIO5 ─────────── HC-SR04 TRIG        3V3 out, read fine as HIGH

   HC-SR04 ECHO ──[1kΩ]──┬──[2kΩ]── GND
                         │
                   ESP32 GPIO18                 5V in, divided to 3.3V

   GND ─────────────────── HC-SR04 GND
```

|From|To|Note|
|---|---|---|
|ESP32 `5V`/VIN pin|HC-SR04 VCC|it needs 5V to work reliably|
|HC-SR04 GND|ESP32 GND|-|
|ESP32 GPIO5|HC-SR04 TRIG|3.3V out is fine, no shifting needed|
|HC-SR04 ECHO|1kΩ resistor leg 1|**this is the dangerous one**|
|1kΩ resistor leg 2|ESP32 GPIO18 and 2kΩ leg 1|-|
|2kΩ resistor leg 2|ESP32 GND|-|

Note only ONE of the two signal lines needs shifting. TRIG is an output from the ESP32 going into the sensor, which is the safe direction. ECHO is an output from the sensor coming back, which is the dangerous one.

That asymmetry is a good habit to build: look at each wire and ask which end is driving it.

UART between a 5V board and an ESP32

Serial connections cross over, and only one of the two directions needs shifting.

|From|To|Shifting needed?|
|---|---|---|
|ESP32 TX (GPIO17)|5V board RX|No, 3.3V is read as HIGH|
|5V board TX|ESP32 RX (GPIO16)|**Yes, divider or shifter**|
|Both GND|Both GND|Always|

What happens if you get it wrong

|Damage|Symptom|
|---|---|
|Immediate|That GPIO stops working, reads stuck or floating|
|Gradual|Pin works for weeks then fails intermittently|
|Collateral|Current into the 3.3V rail can affect other pins|
|Best case|Nothing, you got lucky, but you still overstressed it|

The gradual case is the worst. The protection diode conducts, the pin survives, and everything appears fine. Weeks later that one pin behaves oddly and you have no idea why, because the wiring has not changed.

The mistake everyone makes

Connecting the HC-SR04's ECHO pin straight to a GPIO. It works. The distance readings are correct. And you are dumping 5V into a 3.3V pin every measurement cycle, thousands of times an hour. Fit the two resistors, or buy the HC-SR04P.

Second: assuming you need level shifting in both directions. Usually only one direction is a problem, and shifting the safe direction unnecessarily adds parts and slows the signal.

Third: using a plain 74HC125 instead of a 74**HCT**125 for 3.3V to 5V. The HC version has a higher input threshold and will not reliably see 3.3V as HIGH.

Fourth: forgetting the shared ground. A level shifter with no common ground reference does nothing useful.

Fifth: putting a TXB0104 on an I2C bus. It has active drivers that fight I2C's open-drain pull-ups. Use a BSS138-type shifter for I2C.
