Serial is two wires sending bytes one bit at a time, and it is your only real window into what the board is doing.

Printing

```cpp
/*
1. Serial.begin sets the speed, both ends must agree
2. print stays on the line, println adds a newline
3. the small delay stops the monitor being flooded faster than it can draw
*/

void setup() {
    Serial.begin(115200);
    delay(100);                 // give USB a moment before the first print
    Serial.println("booted");
}

int count = 0;

void loop() {
    Serial.print("count = ");
    Serial.println(count);
    count++;
    delay(1000);
}

// output: booted
//         count = 0
//         count = 1
//         count = 2
```

printf works too, and is nicer for mixed values

```cpp
float temp = 21.475;
int   id   = 3;

Serial.printf("sensor %d = %.2f C\n", id, temp);

// output: sensor 3 = 21.47 C
```

Printing in other bases

```cpp
int x = 200;

Serial.println(x);          // output: 200
Serial.println(x, HEX);     // output: C8
Serial.println(x, BIN);     // output: 11001000
Serial.println(x, DEC);     // output: 200
```

Reading typed input

```cpp
/*
1. Serial.available() says how many bytes are waiting, 0 means nothing typed
2. never read without checking, readString on an empty buffer just blocks
3. trim() removes the newline the monitor sends with your line
*/

void setup() {
    Serial.begin(115200);
    Serial.println("type on or off");
    pinMode(13, OUTPUT);
}

void loop() {
    if (Serial.available() > 0) {
        String cmd = Serial.readStringUntil('\n');
        cmd.trim();

        if (cmd == "on") {
            digitalWrite(13, HIGH);
            Serial.println("led on");
        } else if (cmd == "off") {
            digitalWrite(13, LOW);
            Serial.println("led off");
        } else {
            Serial.println("unknown: " + cmd);
        }
    }
}

// output: type on or off
//         led on
//         unknown: hello
```

The three UARTs

|Port|Object|Default pins|Use for|
|---|---|---|---|
|UART0|`Serial`|TX 1, RX 3|USB serial monitor, leave it alone|
|UART1|`Serial1`|remappable|a second device|
|UART2|`Serial2`|TX 17, RX 16|GPS, RFID readers, another MCU|

Talking to a second device

```cpp
/*
1. TX on one side goes to RX on the other, crossed over
2. GND must be shared or nothing works
3. you can remap the pins in begin(), the ESP32 has a pin matrix
*/

void setup() {
    Serial.begin(115200);                    // to the computer
    Serial2.begin(9600, SERIAL_8N1, 16, 17); // baud, config, RX pin, TX pin
}

void loop() {
    while (Serial2.available()) {
        Serial.write(Serial2.read());        // pass the GPS through to the monitor
    }
}
```

```
  wiring two boards

   ESP32                      device
   ┌───────┐                 ┌───────┐
   │ TX 17 ├─────────────────┤ RX    │
   │ RX 16 ├─────────────────┤ TX    │   crossed
   │ GND   ├─────────────────┤ GND   │   always
   └───────┘                 └───────┘

  If the device is 5V, its TX will kill the ESP32's RX pin.
  Divider or level shifter on that line.
```

What a byte actually looks like on the wire

```
  8N1 = 8 data bits, No parity, 1 stop bit

  idle  start   b0  b1  b2  b3  b4  b5  b6  b7   stop  idle
  ─────┐     ┌───┐       ┌───────┐           ┌────────────
       │     │   │       │       │           │
       └─────┘   └───────┘       └───────────┘

        ├────┤
       one bit time = 1 / baud
       at 115200 that is 8.7 microseconds
```

Baud rate mismatch

```
  Both ends must sample at the same rate. If they don't, the receiver
  reads bits at the wrong moments and you get consistent garbage:

    Serial.begin(115200) with the monitor set to 9600

    output:  ÿÿÿ<Â¾ãÿ¾<ãÿÿ

  Same symptom, three different causes:

    monitor speed wrong            -> change the dropdown
    Serial.begin speed wrong       -> change the code
    only some characters garbled   -> bad ground, or a too-long wire

  The boot ROM also prints its own log at 115200 before your code runs,
  so if the first line is readable gibberish then readable text, that's normal.
```

The gotcha

```
  Serial.print is slow. At 115200 baud each character takes ~87
  microseconds. A println inside a tight loop will completely change
  the timing of your program, and printing inside an interrupt handler
  can hang the board outright, see [[Interrupts (ESP32)]].

  Don't use GPIO 1 and 3 for anything else, they are UART0 and the
  serial monitor plus the uploader both need them.
```
