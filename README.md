# V2Q

## Web-App
[Here](https://paladinic.github.io/esp32_test/) you can access the web-app used to control the ESP32 below.

## Circuit

Button
- One side of the button → GPIO 13
- The other side of the button → GND.

MOSFET (Low-Side Switching)
- Drain → Atomizer coil negative.
- Source → Battery negative (also ESP32 GND).
- Gate → 220 Ω resistor → GPIO 27; plus 10 kΩ from Gate to Source.

Coil
- Coil positive → Battery +.
- Coil negative → MOSFET Drain.

Power
- ESP32 powered via USB (5 V → onboard 3.3 V regulator).
- Battery negative → ESP32 GND (common ground).


## ESP32 Code
```
/*
 * ESP32 BLE + Button Press Counter + Press Limit
 * ----------------------------------------------
 * - Button on GPIO 13 with internal pull-up:
 *     Press = LOW, Not Pressed = HIGH
 * - MOSFET Gate on GPIO 27
 * - BLE Service & Characteristics:
 *     Service UUID       : 00001234-0000-1000-8000-00805f9b34fb
 *     Press Limit (R/W)  : 00003456-0000-1000-8000-00805f9b34fb
 *     Reset (W)          : 00003457-0000-1000-8000-00805f9b34fb
 *
 * When the button transitions from HIGH->LOW, increment pressCount.
 * If pressCount <= pressLimit, MOSFET can turn on with button press.
 * If pressCount > pressLimit, MOSFET stays off.
 * The 'reset' characteristic can reset pressCount to 0.
 */

#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

// Pin assignments
const int BUTTON_PIN      = 13;
const int MOSFET_GATE_PIN = 27;

// Press logic
int pressCount  = 0;
int pressLimit  = 5;   // Default limit; can be changed via BLE
int lastButtonState = HIGH;

// BLE UUIDs
#define SERVICE_UUID            "00001234-0000-1000-8000-00805f9b34fb"
#define CHAR_PRESS_LIMIT_UUID   "00003456-0000-1000-8000-00805f9b34fb"
#define CHAR_RESET_UUID         "00003457-0000-1000-8000-00805f9b34fb"

// Forward declare characteristics so we can update from callbacks
BLECharacteristic *pressLimitCharacteristic;
BLECharacteristic *resetCharacteristic;

// Callback class for Press Limit characteristic (read/write)
class PressLimitCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* characteristic) override {
    std::string value = characteristic->getValue();
    if (!value.empty()) {
      // We expect an integer in the range [1..10]
      int newLimit = atoi(value.c_str());
      // Basic clamp to 1..10
      if (newLimit < 1)  newLimit = 1;
      if (newLimit > 10) newLimit = 10;
      pressLimit = newLimit;
      Serial.printf(">>> New Press Limit set via BLE: %d\n", pressLimit);
    }
  }
};

// Callback class for Reset characteristic (write only)
class ResetCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* characteristic) override {
    std::string value = characteristic->getValue();
    if (!value.empty()) {
      // If any value is written, we reset pressCount
      pressCount = 0;
      Serial.println(">>> pressCount has been RESET via BLE!");
    }
  }
};

void setup() {
  Serial.begin(115200);
  Serial.println("ESP32 BLE + Button Press Counter Starting...");

  // Pin setup
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(MOSFET_GATE_PIN, OUTPUT);
  digitalWrite(MOSFET_GATE_PIN, LOW);

  // --- Setup BLE ---
  BLEDevice::init("ESP32-Atomizer");  // The name shown during scanning
  BLEServer *pServer = BLEDevice::createServer();

  // Create a BLE service
  BLEService *pService = pServer->createService(SERVICE_UUID);

  // Create the Press Limit characteristic (read/write)
  pressLimitCharacteristic = pService->createCharacteristic(
    CHAR_PRESS_LIMIT_UUID,
    BLECharacteristic::PROPERTY_READ   |
    BLECharacteristic::PROPERTY_WRITE
  );
  pressLimitCharacteristic->setCallbacks(new PressLimitCallbacks());
  // We store the default limit as a string for read
  pressLimitCharacteristic->setValue(String(pressLimit).c_str());

  // Create the Reset characteristic (write only)
  resetCharacteristic = pService->createCharacteristic(
    CHAR_RESET_UUID,
    BLECharacteristic::PROPERTY_WRITE
  );
  resetCharacteristic->setCallbacks(new ResetCallbacks());

  // Start the service
  pService->start();

  // Start advertising
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(true);
  pAdvertising->start();
  
  Serial.println("BLE Service & Characteristics are now advertising...");
}

void loop() {
  // Read the button state
  int buttonState = digitalRead(BUTTON_PIN);

  // Check for a new press transition: HIGH->LOW
  if (buttonState == LOW && lastButtonState == HIGH) {
    pressCount++;
    Serial.printf("Button pressed, pressCount = %d\n", pressCount);

    // Debounce
    delay(50);
  }
  lastButtonState = buttonState;

  // If button is currently pressed AND pressCount <= pressLimit, MOSFET on
  // Otherwise, keep it off.
  if (buttonState == LOW && pressCount <= pressLimit) {
    digitalWrite(MOSFET_GATE_PIN, HIGH);
  } else {
    digitalWrite(MOSFET_GATE_PIN, LOW);
  }

  // Update the characteristic’s read value so the client sees the latest limit
  pressLimitCharacteristic->setValue(String(pressLimit).c_str());

  delay(20);
}
```
