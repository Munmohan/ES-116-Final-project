# ES-116-Final-project
#ifdef ENABLE_DEBUG
#define DEBUG_ESP_PORT Serial
#define NODEBUG_WEBSOCKETS
#define NDEBUG
#endif

// Required libraries
#include <Arduino.h>
#include <ESP8266WiFi.h>
#include "SinricPro.h"
#include "SinricProSwitch.h"
#include <map>

// WiFi credentials
#define WIFI_SSID "Nothing Phone (2a)_5263"
#define WIFI_PASS "123456789"

// SinricPro credentials
#define APP_KEY "346ba865-b29b-4d87-a492-68e11ce6218f"
#define APP_SECRET "f4f972fb-7bc0-48d4-ac5c-a6b9e1c0a540-d08e0467-0d2d-4306-bcfa-2c7671fef57b"

// Device IDs from Sinric Pro
#define device_ID_1 "680374fe947cbabd20016219"
#define device_ID_2 "680374458ed485694c1cb430"
#define device_ID_3 "680374melodydeviceid123"  // Melody buzzer device
#define device_ID_4 "SWITCH_ID_NO_4_HERE"

// Pin mapping for relays and switches
#define RelayPin1 5   // D1
#define RelayPin2 4   // D2
#define RelayPin4 12  // D6

#define SwitchPin1 10 // SD3
#define SwitchPin2 0  // D3
#define SwitchPin4 3  // RX

#define BUZZER_PIN 14 // D5 (previously RelayPin3)
#define wifiLed 16    // D0 (WiFi status LED)

#define TACTILE_BUTTON 1
#define BAUD_RATE 9600
#define DEBOUNCE_TIME 250 // ms

// Structure for holding relay and switch pin pair
typedef struct {
  int relayPIN;
  int flipSwitchPIN;
} deviceConfig_t;

// Mapping device IDs to their relay and switch pins
std::map<String, deviceConfig_t> devices = {
  {device_ID_1, {RelayPin1, SwitchPin1}},
  {device_ID_2, {RelayPin2, SwitchPin2}},
  // Melody handled separately
  {device_ID_4, {RelayPin4, SwitchPin4}}
};

// Structure to handle flip switch state and debounce
typedef struct {
  String deviceId;
  bool lastFlipSwitchState;
  unsigned long lastFlipSwitchChange;
} flipSwitchConfig_t;

std::map<int, flipSwitchConfig_t> flipSwitches;

// Set relay pins as OUTPUT and turn them OFF initially
void setupRelays() {
  for (auto &device : devices) {
    int relayPIN = device.second.relayPIN;
    pinMode(relayPIN, OUTPUT);
    digitalWrite(relayPIN, HIGH); // HIGH = OFF for active-low relay
  }
}

// Initialize all switches with INPUT_PULLUP
void setupFlipSwitches() {
  for (auto &device : devices) {
    flipSwitchConfig_t flipSwitchConfig;
    flipSwitchConfig.deviceId = device.first;
    flipSwitchConfig.lastFlipSwitchChange = 0;
    flipSwitchConfig.lastFlipSwitchState = true;
    int flipSwitchPIN = device.second.flipSwitchPIN;
    flipSwitches[flipSwitchPIN] = flipSwitchConfig;
    pinMode(flipSwitchPIN, INPUT_PULLUP);
  }
}

// Function to play a simple melody on buzzer
void playMelody() {
  int melody[] = { 262, 294, 330, 349, 392, 440, 494, 523 }; // Notes C to C
  int durations[] = { 4, 4, 4, 4, 4, 4, 4, 4 }; // Each note duration

  for (int i = 0; i < 8; i++) {
    int duration = 1000 / durations[i];
    tone(BUZZER_PIN, melody[i], duration);
    delay(duration * 1.3);
    noTone(BUZZER_PIN);
  }
}

// Alexa callback function for controlling devices
bool onPowerState(String deviceId, bool &state) {
  Serial.printf("%s: %s\r\n", deviceId.c_str(), state ? "on" : "off");

  // If device is the buzzer, play melody
  if (deviceId == device_ID_3) {
    if (state) playMelody();
    return true;
  }

  // Otherwise, toggle relay
  int relayPIN = devices[deviceId].relayPIN;
  digitalWrite(relayPIN, !state); // Active-low logic
  return true;
}

// Handle flip switch presses with debounce
void handleFlipSwitches() {
  unsigned long actualMillis = millis();
  for (auto &flipSwitch : flipSwitches) {
    unsigned long lastChange = flipSwitch.second.lastFlipSwitchChange;
    if (actualMillis - lastChange > DEBOUNCE_TIME) {
      int pin = flipSwitch.first;
      bool lastState = flipSwitch.second.lastFlipSwitchState;
      bool currentState = digitalRead(pin);

      if (currentState != lastState) {
#ifdef TACTILE_BUTTON
        if (currentState) { // Only trigger on button release
#endif
          flipSwitch.second.lastFlipSwitchChange = actualMillis;
          String deviceId = flipSwitch.second.deviceId;
          int relayPIN = devices[deviceId].relayPIN;
          bool newState = !digitalRead(relayPIN); // Toggle relay
          digitalWrite(relayPIN, newState);
          SinricProSwitch &mySwitch = SinricPro[deviceId];
          mySwitch.sendPowerStateEvent(!newState); // Update SinricPro
#ifdef TACTILE_BUTTON
        }
#endif
        flipSwitch.second.lastFlipSwitchState = currentState;
      }
    }
  }
}

// Connect to WiFi and light LED on success
void setupWiFi() {
  Serial.printf("\r\n[Wifi]: Connecting");
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    Serial.printf(".");
    delay(250);
  }
  digitalWrite(wifiLed, LOW); // Turn on LED to show connected
  Serial.printf("connected!\r\n[WiFi]: IP-Address is %s\r\n", WiFi.localIP().toString().c_str());
}

// Initialize SinricPro devices and callbacks
void setupSinricPro() {
  for (auto &device : devices) {
    const char *deviceId = device.first.c_str();
    SinricProSwitch &mySwitch = SinricPro[deviceId];
    mySwitch.onPowerState(onPowerState);
  }

  // Special device: buzzer
  SinricProSwitch &melodyDevice = SinricPro[device_ID_3];
  melodyDevice.onPowerState(onPowerState);

  SinricPro.begin(APP_KEY, APP_SECRET);
  SinricPro.restoreDeviceStates(true);
}

// Main setup function
void setup() {
  Serial.begin(BAUD_RATE);
  pinMode(wifiLed, OUTPUT);
  digitalWrite(wifiLed, HIGH); // Initially off

  pinMode(BUZZER_PIN, OUTPUT); // Setup buzzer pin
  setupRelays();
  setupFlipSwitches();
  setupWiFi();
  setupSinricPro();
}

// Main loop function
void loop() {
  SinricPro.handle(); // Keep SinricPro connection alive
  handleFlipSwitches(); // Check for tactile switch input
}
