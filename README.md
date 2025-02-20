# V2Q

## Web-App
[Here](https://paladinic.github.io/esp32_test/) you can access the web-app used to control the ESP32 below.

## Circuit

Design and instructions also found [here](https://docs.google.com/presentation/d/1D4ElymJ7oJMBcLeamXaluF73_FnM6GacjfgMBf-IujM/edit#slide=id.g322d3f5ab80_0_0).

Button
- One side of the button → BUTTON Pin on board 
- The other side of the button → GND.

MOSFET (Low-Side Switching)
- Drain → Atomizer coil negative.
- Source → Battery negative (also ESP32 GND).
- Gate → 220 Ω resistor → MOSFET Pin on board ; plus 10 kΩ from Gate to Source.

Coil
- Coil positive → Battery +.
- Coil negative → MOSFET Drain.

Power
- ESP32 powered via USB (5 V → onboard 3.3 V regulator).
- Battery negative → ESP32 GND (common ground).

## Arduino Code:

Below you can find 2 versions of the Arduino code, with different dependencies for Bluetooth. 

### For Arduino Cloud IDE

```cpp
#include <ArduinoBLE.h>

// Pin assignments
const int BUTTON_PIN      = 13; // Adjust for your board
const int MOSFET_GATE_PIN = 27; // Adjust for your board

// Press logic
int pressCount    = 0;
int pressLimit    = 5;    // Default limit
int lastButtonState = HIGH;

// BLE UUIDs (same as your original code)
#define SERVICE_UUID            "00001234-0000-1000-8000-00805f9b34fb"
#define CHAR_PRESS_LIMIT_UUID   "00003456-0000-1000-8000-00805f9b34fb"
#define CHAR_RESET_UUID         "00003457-0000-1000-8000-00805f9b34fb"

// Create a BLE Service
BLEService pressService(SERVICE_UUID);

// Create Characteristics
// - For the limit, we'll use an integer characteristic (BLEIntCharacteristic).
// - For the reset, we can use a simple byte characteristic, and treat any non-zero write as a reset trigger.
BLEIntCharacteristic  pressLimitCharacteristic(CHAR_PRESS_LIMIT_UUID,  BLERead | BLEWrite);
BLEByteCharacteristic resetCharacteristic(CHAR_RESET_UUID, BLEWrite);

void setup() {
  Serial.begin(115200);
  Serial.println("Starting BLE (ArduinoBLE) + Button Press Counter...");

  // Setup pins
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(MOSFET_GATE_PIN, OUTPUT);
  digitalWrite(MOSFET_GATE_PIN, LOW);

  // Initialize the ArduinoBLE library
  if (!BLE.begin()) {
    Serial.println("Starting BLE failed!");
    while (1);
  }

  // Set advertised local name
  BLE.setLocalName("ESP32-Atomizer");  // Might say "Nano-Atomizer" if using an Arduino board
  BLE.setAdvertisedService(pressService);

  // Add characteristics to the service
  pressService.addCharacteristic(pressLimitCharacteristic);
  pressService.addCharacteristic(resetCharacteristic);

  // Add the service
  BLE.addService(pressService);

  // Initialize characteristic values
  pressLimitCharacteristic.writeValue(pressLimit);

  // Start advertising
  BLE.advertise();
  Serial.println("BLE Service & Characteristics are now advertising...");
}

void loop() {
  // Continuously poll BLE for events
  BLE.poll();

  // Handle characteristic writes
  handleCharacteristicUpdates();

  // Read the button state
  int buttonState = digitalRead(BUTTON_PIN);

  // Check for a new press transition: HIGH -> LOW
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
  pressLimitCharacteristic.writeValue(pressLimit);

  delay(20);
}

void handleCharacteristicUpdates() {
  // If pressLimitCharacteristic has been written to by the client
  if (pressLimitCharacteristic.written()) {
    int newLimit = pressLimitCharacteristic.value();
    // Basic clamp to [1..10]
    if (newLimit < 1)  newLimit = 1;
    if (newLimit > 10) newLimit = 10;
    pressLimit = newLimit;
    Serial.printf(">>> New Press Limit set via BLE: %d\n", pressLimit);
  }

  // If resetCharacteristic has been written to by the client
  if (resetCharacteristic.written()) {
    byte resetVal = resetCharacteristic.value();
    // If any non-zero value is written, reset
    if (resetVal != 0) {
      pressCount = 0;
      Serial.println(">>> pressCount has been RESET via BLE!");
    }
  }
}

```

### For Arduino IDE

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

// Pin assignments
const int BUTTON_PIN      = 20;
const int MOSFET_GATE_PIN = 21;

// Press logic
int pressCount  = 0;
int pressLimit  = 5;   // Default limit; can be changed via BLE
int lastButtonState = HIGH;

// BLE UUIDs
#define SERVICE_UUID            "00001234-0000-1000-8000-00805f9b34fb"
#define CHAR_PRESS_LIMIT_UUID   "00003456-0000-1000-8000-00805f9b34fb"
#define CHAR_RESET_UUID         "00003457-0000-1000-8000-00805f9b34fb"

// Forward declare characteristics so we can update them in callbacks
BLECharacteristic *pressLimitCharacteristic;
BLECharacteristic *resetCharacteristic;

// Callback class for Press Limit characteristic (read/write)
class PressLimitCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* characteristic) override {
    // Read the incoming data as a std::string
    std::string value = characteristic->getValue();
    if (!value.empty()) {
      // Convert to integer
      int newLimit = atoi(value.c_str());
      // Clamp between 1..10
      if (newLimit < 1)  newLimit = 1;
      if (newLimit > 10) newLimit = 10;
      
      // Update global pressLimit
      pressLimit = newLimit;
      Serial.printf(">>> New Press Limit set via BLE: %d\n", pressLimit);
    }
  }
};

// Callback class for Reset characteristic (write only)
class ResetCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* characteristic) override {
    // Read the incoming data (any data triggers a reset)
    std::string value = characteristic->getValue();
    if (!value.empty()) {
      // Reset pressCount
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
  BLEDevice::init("ESP32-Atomizer");    // The name shown during scanning
  BLEServer *pServer = BLEDevice::createServer();

  // Create a BLE service
  BLEService *pService = pServer->createService(SERVICE_UUID);

  // Create the Press Limit characteristic (read/write)
  pressLimitCharacteristic = pService->createCharacteristic(
    CHAR_PRESS_LIMIT_UUID,
    BLECharacteristic::PROPERTY_READ |
    BLECharacteristic::PROPERTY_WRITE
  );
  pressLimitCharacteristic->setCallbacks(new PressLimitCallbacks());
  
  // Set an initial default value (needs .c_str() to avoid compiler errors)
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
  // Otherwise, keep it off
  if (buttonState == LOW && pressCount <= pressLimit) {
    digitalWrite(MOSFET_GATE_PIN, HIGH);
  } else {
    digitalWrite(MOSFET_GATE_PIN, LOW);
  }

  // Continuously update the read value so the BLE client sees the latest limit
  pressLimitCharacteristic->setValue(String(pressLimit).c_str());

  delay(20);
}
```
