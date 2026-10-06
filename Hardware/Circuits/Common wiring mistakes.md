Almost every hardware problem you will hit is one of about twenty things, and this is the list to work through before you start rewriting code.

The rule that saves the most time: **if it is hardware, the code will not fix it.** When something does not work, spend two minutes on the table below before you touch the sketch.

Nothing works at all

|Symptom|Likely cause|Fix|
|---|---|---|
|Circuit completely dead|No common ground between supplies|Physically join every GND. This is the number one cause|
|Circuit completely dead|Power rail not connected|Jumper 3V3 and GND from the ESP32 to the rails|
|Half the board dead|Breadboard rails have a mid-board break|Bridge the gap with two jumpers, see [[Breadboards and wiring]]|
|Nothing on the serial monitor|Wrong baud rate|Set 115200|
|Nothing on the serial monitor|Charge-only USB cable|Try a different cable, this is very common|
|Board not detected by the IDE|Missing CH340 or CP2102 driver|Install the USB serial driver|
|Board resets constantly at boot|GPIO12 held HIGH, or GPIO0 held LOW|Free up the strapping pins|

LED problems

|Symptom|Likely cause|Fix|
|---|---|---|
|LED does nothing|Fitted backwards|Flip it. Long leg to positive, flat side to GND|
|LED does nothing|Both legs in the same breadboard column|Components must span two columns|
|LED does nothing|Loose wire|Reseat every jumper|
|LED very dim|Resistor far too large|220Ω on 3.3V, not 10kΩ|
|Blue/white LED barely lights on 3V3|Forward voltage is ~3.2V, no headroom|Use 5V, or use a red/green LED|
|LED bright then dead|No resistor|Always fit one, see [[LED current limiting circuit]]|
|One LED bright, others dim|Several LEDs sharing one resistor|One resistor per LED|
|LED flickers|Loose contact, or a floating output pin|Reseat, and set `pinMode(pin, OUTPUT)`|

Button problems

|Symptom|Likely cause|Fix|
|---|---|---|
|Reads pressed permanently|Two legs from the same side of a 4-pin tactile switch|Rotate the switch 90°, see [[Buttons and Switches]]|
|Random triggers, reacts to your hand|Floating pin, no pull-up|`pinMode(pin, INPUT_PULLUP)`|
|`INPUT_PULLUP` has no effect|Pin is GPIO34-39, which have no internal resistors|Move pins, or add a real 10kΩ|
|One press counts as three to five|Contact bounce|Debounce it, see [[Button debouncing]]|
|Logic seems backwards|Pull-up means pressed = LOW|Test for `== LOW`|
|Triggers when a motor starts|Electrical noise|Decoupling capacitors, separate supplies|
|Button works, but only sometimes|Worn breadboard contact|Move to a different row|

Power problems

|Symptom|Likely cause|Fix|
|---|---|---|
|`Brownout detector was triggered`|Supply cannot deliver the current|Better cable, better supply, 470µF capacitor|
|ESP32 resets when a motor starts|Motor on the same supply as logic|Separate supply, common ground, bulk capacitor|
|ESP32 resets when a servo moves|Servo powered from the `3V3` or `5V` pin|Give the servo its own 5V supply|
|Board resets when WiFi connects|Radio current spike sags the rail|100µF across 3V3 and GND, better USB cable|
|Regulator too hot to touch|`(Vin - Vout) × I` is too high|Use a buck converter, see [[Voltage Regulators]]|
|Battery flat in hours|Something drawing current constantly|Check for a low-value voltage divider or an always-on LED|
|Motor weak, driver hot|L298N drops ~2V, or supply too weak|Use a DRV8833, or feed 2V higher|
|Voltage reads lower than expected|Protection diode dropping 0.7V|Use a Schottky, or account for it|

Sensor problems

|Symptom|Likely cause|Fix|
|---|---|---|
|I2C sensor not found|Wiring, power, or wrong pins|Run the I2C scanner sketch first, always|
|I2C sensor not found|SDA and SCL swapped|SDA is GPIO21, SCL is GPIO22|
|Analogue reading works, then dies on WiFi|Using an ADC2 pin|Move to GPIO32-39, which are ADC1|
|Analogue reading jumps around|ESP32 ADC is noisy|Average 16 samples|
|Analogue reading pinned at 4095|Input above 3.1V, or floating|Add a divider, or check the wiring|
|Ultrasonic returns 0 or garbage|`pulseIn` timeout, or object too close/far|Check TRIG pulse, range is 2cm to 4m|
|Sensor readings drift when motor runs|Motor noise on the shared ground|Separate ground return, 100nF at the motor|
|UART device silent|TX and RX not crossed|Module TX to ESP32 RX, and vice versa|
|5V sensor output kills the pin over time|No level shifting|Divider or HC-SR04P, see [[Level shifting 3V3 and 5V]]|

Motor and driver problems

|Symptom|Likely cause|Fix|
|---|---|---|
|Driver completely dead|`SLP` / `STBY` pin floating|Pull it HIGH to 3V3|
|Motor buzzes, does not turn|PWM duty too low|Start from ~60/255, not 0|
|Motor buzzes, does not turn|Stepper coil pairs wired wrong|Find pairs with a multimeter|
|Motor only turns one way|One input pin not connected or not PWM configured|Check both inputs|
|MOSFET hot, motor weak|Not a logic-level MOSFET on 3.3V|IRF540N needs 10V. Use an IRLB8721|
|Motor spins briefly on every reset|No gate pull-down|10kΩ from gate to GND|
|Transistor died|No flyback diode|Fit one, stripe to positive, see [[Flyback diode protection]]|
|Stepper driver died|No 100µF on VMOT, or motor unplugged while powered|Capacitor, and never hot-unplug|
|Stepper loses position over time|Missed steps, accelerating too hard|Slow down, use AccelStepper|
|Servo jitters constantly|Noisy supply, or no common ground|470µF, check the ground|

Damaged pin problems

|Symptom|Likely cause|Fix|
|---|---|---|
|One pin stuck HIGH or LOW|5V applied to it previously|That pin is gone, move to another|
|Pin reads noise regardless of code|Pin damaged, or floating input|Test with a known-good pin first|
|Board works but one function fails|Damaged GPIO|Reassign in code, use a different pin|

The general debugging method

Do these in order. Do not skip to the code.

|Step|Action|
|---|---|
|1|Power off. Check for shorts, especially 3V3 to GND|
|2|Multimeter continuity: is ESP32 GND actually joined to every other ground?|
|3|Multimeter voltage: is 3V3 really 3.3V, is 5V really 5V?|
|4|Reseat every jumper wire. Loose contacts are the top cause of intermittent faults|
|5|Simplify. Disconnect everything except the one thing that is failing|
|6|Test the pin with a bare LED and resistor. Does the pin work at all?|
|7|Test the component separately, straight to power. Does the component work at all?|
|8|Only now, look at the code|

Step 5 is the one people resist and it is the most effective. Half a broken circuit is much easier to reason about than all of it.

Things that look like software bugs but are not

|Looks like|Actually is|
|---|---|
|Random crashes or reboots|Power brownout|
|Counter incrementing by 5|Button bounce|
|Sensor returning nonsense during motion|Electrical noise from a motor|
|Code working then not, with no changes|Loose breadboard wire|
|ADC returning 0 after WiFi connects|ADC2 pin|
|Interrupt firing repeatedly|Bounce, or a floating pin|
|Board not entering the loop|Brownout, or a strapping pin held wrong|
|Works on USB, fails on battery|Battery cannot deliver the current|

The ESP32 pins to avoid

|Pin|Problem|
|---|---|
|GPIO6, 7, 8, 9, 10, 11|Wired to internal SPI flash. Using them crashes the board|
|GPIO0|Boot mode. LOW at reset enters flash mode|
|GPIO2|Must not be held HIGH at boot on some boards|
|GPIO12|HIGH at boot sets the wrong flash voltage, board may not start|
|GPIO15|LOW at boot suppresses the boot log|
|GPIO34-39|Input only. No output, no internal pull-up or pull-down|
|GPIO0, 2, 4, 12-15, 25-27|ADC2, unusable for analogue while WiFi is on|

Safe general-purpose pins: **4, 5, 13, 14, 16, 17, 18, 19, 21, 22, 23, 25, 26, 27, 32, 33**.

Safe analogue pins with WiFi on: **32, 33, 34, 35, 36, 39**.

Habits that prevent most of this

|Habit|Prevents|
|---|---|
|Black wire is always ground, red is always positive|Connecting 5V to a GPIO|
|Connect grounds first, before anything else|The number one dead-circuit cause|
|Power off while rewiring|Momentary shorts as a wire brushes past|
|Check polarity twice on electrolytics and diodes|Popped capacitors, shorted supplies|
|Multimeter continuity check before powering up|Basically everything|
|Build one section at a time and test it|Debugging a whole broken system at once|
|Write down which pin does what|Wiring to the wrong pin and blaming the code|

The mistake everyone makes

Assuming it is the code. It usually is not. When something worked yesterday and does not today and you have not changed anything, the answer is a loose wire, not a compiler bug.

Second: changing three things at once, then not knowing which fixed it. Change one thing, test, repeat.

Third: not owning a multimeter. A £10 meter with a continuity beeper turns most of the table above into a five-second check. It is the single best purchase in this whole section.
