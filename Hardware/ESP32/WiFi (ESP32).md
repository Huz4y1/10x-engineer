WiFi is the reason to pick an ESP32 over almost anything else, and it's about ten lines to get onto a network.

Connecting

```cpp
/*
1. WIFI_STA means station mode, joining someone else's network
2. begin() is non-blocking, so you wait in a loop until the status changes
3. localIP() is the address to type into a browser later
*/

#include <WiFi.h>

const char *ssid = "MyNetwork";
const char *pass = "password123";

void setup() {
    Serial.begin(115200);

    WiFi.mode(WIFI_STA);
    WiFi.begin(ssid, pass);

    Serial.print("connecting");

    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }

    Serial.println();
    Serial.print("IP: ");
    Serial.println(WiFi.localIP());
    Serial.print("RSSI: ");
    Serial.println(WiFi.RSSI());
}

void loop() {}

// output: connecting.....
//         IP: 192.168.1.42
//         RSSI: -58
```

Status codes

|Value|Meaning|
|---|---|
|`WL_CONNECTED`|On the network|
|`WL_NO_SSID_AVAIL`|Network not found — typo, or it's 5 GHz|
|`WL_CONNECT_FAILED`|Wrong password|
|`WL_IDLE_STATUS`|Still trying|
|`WL_DISCONNECTED`|Dropped, or `begin()` hasn't been called|

Signal strength, in RSSI

|RSSI|Quality|
|---|---|
|-30 to -50|Excellent|
|-50 to -60|Good|
|-60 to -70|Workable|
|below -80|Expect dropouts|

Scanning for networks

```cpp
void setup() {
    Serial.begin(115200);
    WiFi.mode(WIFI_STA);
    WiFi.disconnect();

    int n = WiFi.scanNetworks();

    for (int i = 0; i < n; i++) {
        Serial.printf("%s  %d dBm  %s\n",
            WiFi.SSID(i).c_str(),
            WiFi.RSSI(i),
            WiFi.encryptionType(i) == WIFI_AUTH_OPEN ? "open" : "locked");
    }
}

// output: MyNetwork  -58 dBm  locked
//         BTHub-7A2  -81 dBm  locked
```

An HTTP GET

```cpp
/*
1. HTTPClient does the whole request for you
2. the return is the HTTP status code, 200 is fine, negative means it never connected
3. always call end(), it frees the socket
*/

#include <WiFi.h>
#include <HTTPClient.h>

void fetch() {
    if (WiFi.status() != WL_CONNECTED) return;

    HTTPClient http;
    http.begin("http://worldtimeapi.org/api/timezone/Europe/London");

    int code = http.GET();

    if (code == 200) {
        String body = http.getString();
        Serial.println(body);
    } else {
        Serial.printf("failed: %d\n", code);
    }

    http.end();
}

// output: {"abbreviation":"BST","datetime":"2026-07-21T14:02:11.5+01:00", ... }
```

A POST with JSON

```cpp
HTTPClient http;
http.begin("http://example.com/api/readings");
http.addHeader("Content-Type", "application/json");

String body = "{\"temp\":21.4,\"id\":3}";
int code = http.POST(body);

Serial.println(code);      // output: 201
http.end();
```

Serving a tiny web page to control the board

```cpp
/*
1. WebServer listens on port 80 and calls your function for each URL
2. server.on maps a path to a handler
3. server.handleClient() must run constantly, so no delay() in loop
4. browse to the IP printed at boot and you get buttons
*/

#include <WiFi.h>
#include <WebServer.h>

WebServer server(80);

#define LED 13

void handleRoot() {
    String html =
        "<html><body style='font-family:sans-serif;text-align:center'>"
        "<h2>ESP32</h2>"
        "<p><a href='/on'><button>ON</button></a></p>"
        "<p><a href='/off'><button>OFF</button></a></p>"
        "</body></html>";

    server.send(200, "text/html", html);
}

void handleOn() {
    digitalWrite(LED, HIGH);
    server.sendHeader("Location", "/");    // bounce back to the main page
    server.send(303);
}

void handleOff() {
    digitalWrite(LED, LOW);
    server.sendHeader("Location", "/");
    server.send(303);
}

void setup() {
    Serial.begin(115200);
    pinMode(LED, OUTPUT);

    WiFi.begin("MyNetwork", "password123");
    while (WiFi.status() != WL_CONNECTED) delay(500);

    Serial.println(WiFi.localIP());

    server.on("/",    handleRoot);
    server.on("/on",  handleOn);
    server.on("/off", handleOff);
    server.begin();
}

void loop() {
    server.handleClient();     // this must be called constantly
}

// output: 192.168.1.42     ← open that in a browser on the same network
```

Being the access point instead

```cpp
/*
1. softAP makes the ESP32 its own network, no router needed
2. useful for a device out in a field, or for first-time WiFi setup
3. the default address is 192.168.4.1
*/

WiFi.softAP("ESP32-Setup", "12345678");    // password must be 8+ chars
Serial.println(WiFi.softAPIP());           // output: 192.168.4.1
```

Reconnecting, because it will drop

```cpp
void loop() {
    static unsigned long lastCheck = 0;

    if (millis() - lastCheck > 10000) {
        lastCheck = millis();

        if (WiFi.status() != WL_CONNECTED) {
            Serial.println("dropped, reconnecting");
            WiFi.disconnect();
            WiFi.begin(ssid, pass);
        }
    }

    server.handleClient();
}
```

The gotcha

```
  The ESP32 is 2.4 GHz only. A 5 GHz network is simply invisible to
  it, and the symptom is WL_NO_SSID_AVAIL with a name you can see on
  your phone. Split the band or use the 2.4 GHz SSID.

  WiFi kills ADC2, so every analogue pin must be on ADC1 (32-39),
  see [[Analog input and ADC (ESP32)]].

  WiFi pulls 150-250mA in bursts. A weak USB port causes
  "Brownout detector was triggered" and a reboot the moment it connects.

  Don't put credentials in code you'll ever push. Use a secrets.h
  that's gitignored.
```
