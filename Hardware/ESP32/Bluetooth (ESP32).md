The ESP32 does two completely different kinds of Bluetooth and picking the wrong one wastes an afternoon.

The two kinds

|                |Classic Serial (SPP)|BLE|
|---|---|---|
|Feels like|A wireless serial cable|A tiny database the phone reads and writes|
|Setup effort|About 5 lines|Services, characteristics, callbacks|
|Power|Higher|Very low, made for coin cells|
|iPhone support|NO, Apple blocks SPP|Yes|
|Android support|Yes|Yes|
|Flash cost|~1.2 MB|~1.0 MB|
|Use for|Quick debugging, Android app, hobby remote|Anything with an iPhone, or battery powered|

Classic Bluetooth serial, the easy one

```cpp
/*
1. BluetoothSerial behaves exactly like Serial, same print/read API
2. begin() takes the name that shows up in the phone's pairing list
3. pair the phone, then use any "Bluetooth terminal" app
*/

#include "BluetoothSerial.h"

BluetoothSerial BT;

#define LED 13

void setup() {
    Serial.begin(115200);
    pinMode(LED, OUTPUT);

    BT.begin("ESP32-Robot");        // the name phones will see
    Serial.println("pair with ESP32-Robot");
}

void loop() {
    if (BT.available()) {
        char c = BT.read();

        if (c == '1') {
            digitalWrite(LED, HIGH);
            BT.println("led on");
        } else if (c == '0') {
            digitalWrite(LED, LOW);
            BT.println("led off");
        }
    }

    // and it works the other way, phone as a wireless serial monitor
    if (Serial.available()) {
        BT.write(Serial.read());
    }
}

// output on the phone: led on
//                      led off
```

BLE, the one iPhones can talk to

```
  BLE structure, which is the bit that confuses everyone

   Device "ESP32-Sensor"
     └── Service        (a UUID, a group of related values)
           ├── Characteristic  (a UUID, one actual value)
           │     ├── READ      phone can read it
           │     ├── WRITE     phone can change it
           │     └── NOTIFY    device pushes updates unprompted
           └── Characteristic ...

  A characteristic is basically one variable exposed over the air.
  UUIDs are just long unique names, generate them at uuidgen.com.
```

A BLE device the phone can write to

```cpp
/*
1. create a server, a service, then a characteristic inside it
2. the callback fires whenever the phone writes to that characteristic
3. advertising is what makes the device visible to scanners
4. test it with the nRF Connect app, on either phone
*/

#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>

#define SERVICE_UUID "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHAR_UUID    "beb5483e-36e1-4688-b7f5-ea07361b26a8"

#define LED 13

class Handler : public BLECharacteristicCallbacks {
    void onWrite(BLECharacteristic *c) {
        String value = c->getValue().c_str();

        Serial.println(value);

        if (value == "on")  digitalWrite(LED, HIGH);
        if (value == "off") digitalWrite(LED, LOW);
    }
};

void setup() {
    Serial.begin(115200);
    pinMode(LED, OUTPUT);

    BLEDevice::init("ESP32-Light");

    BLEServer  *server  = BLEDevice::createServer();
    BLEService *service = server->createService(SERVICE_UUID);

    BLECharacteristic *ch = service->createCharacteristic(
        CHAR_UUID,
        BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_WRITE);

    ch->setCallbacks(new Handler());
    ch->setValue("off");

    service->start();
    BLEDevice::getAdvertising()->start();     // now it's visible

    Serial.println("advertising");
}

void loop() {
    delay(1000);
}

// output: advertising
//         on
//         off
```

Pushing a sensor value out with NOTIFY

```cpp
/*
1. NOTIFY means the phone subscribes once and the device sends updates itself
2. far more efficient than the phone polling a READ
3. this is how heart-rate straps and thermometers work
*/

BLECharacteristic *tempChar;

void setup() {
    // ... same server/service setup ...

    tempChar = service->createCharacteristic(
        CHAR_UUID,
        BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_NOTIFY);

    tempChar->addDescriptor(new BLE2902());   // required for notify to work
}

void loop() {
    float temp = 21.4;

    char buf[8];
    snprintf(buf, sizeof(buf), "%.1f", temp);

    tempChar->setValue(buf);
    tempChar->notify();          // pushed to any subscribed phone

    delay(2000);
}
```

Knowing when a phone connects

```cpp
class ServerHandler : public BLEServerCallbacks {
    void onConnect(BLEServer *s) {
        Serial.println("phone connected");
    }
    void onDisconnect(BLEServer *s) {
        Serial.println("phone gone");
        BLEDevice::getAdvertising()->start();   // MUST restart, or nobody can rejoin
    }
};

server->setCallbacks(new ServerHandler());
```

The gotcha

```
  Bluetooth and WiFi share ONE 2.4 GHz radio. Both at once works but
  throughput drops and connections get flaky. Prefer one or the other.

  The BT stack is huge. "Sketch too big" is normal, so pick a
  partition scheme with more app space:
    Tools > Partition Scheme > "Huge APP (3MB No OTA)"

  Classic SPP does not work with iPhones at all. If the target is
  an iPhone, it's BLE, no way around it.

  After onDisconnect you must call advertising->start() again,
  otherwise the device vanishes forever and looks broken.

  Deep sleep and Bluetooth don't mix well, the stack takes a second
  or two to come back up, see [[Deep sleep and power saving (ESP32)]].
```
