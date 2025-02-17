# V2Q

## Web-App
[Here](https://paladinic.github.io/esp32_test/) you can access the web-app used to control the ESP32 below.

## Circuit

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


## ESP32 (arduino) Code

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
