#include <NewPing.h>
#include <SoftwareSerial.h>
#include <SPI.h>
#include <Wire.h>

// Pin Definitions
const int soilMoisturePin = A0;        // Soil moisture sensor pin
const int waterPumpPin = 7;            // Water pump relay pin
const int drainagePumpPin = 6;         // Drainage pump relay pin
#define TRIG_PIN 9                     // Ultrasonic sensor trigger pin
#define ECHO_PIN 8                     // Ultrasonic sensor echo pin
#define MAX_DISTANCE 200               // Max distance for ultrasonic sensor

NewPing sonar(TRIG_PIN, ECHO_PIN, MAX_DISTANCE);

// GSM Module
SoftwareSerial gsmSerial(10, 11);      // RX, TX pins for GSM
const char phone_number[] = "+123456789"; // Replace with your phone number

// Battery Management - Low Power Sleep Mode
const int sleepTime = 60000;  // Time for sleep in milliseconds (1 min) to save battery

// AWD Parameters
int soilMoistureValue;
int soilMoisturePercentage;
int waterDepth;
int minimumMoisture = 30;   // Minimum moisture percentage to trigger watering
int maximumWaterDepth = 15; // Max water depth in cm for AWD (Flooding)
int minimumWaterDepth = 5;  // Minimum water depth in cm for draining (Drying)
int floodingDuration = 72 * 3600 * 1000; // Duration for flooding (in milliseconds, 3 days)
int dryingDuration = 72 * 3600 * 1000;   // Duration for drying (in milliseconds, 3 days)
unsigned long lastFloodTime = 0;
unsigned long lastDryTime = 0;

void setup() {
  Serial.begin(9600);
  gsmSerial.begin(9600);
  
  // Pin Setup
  pinMode(waterPumpPin, OUTPUT);
  pinMode(drainagePumpPin, OUTPUT);
  pinMode(soilMoisturePin, INPUT);
  
  // Initialize GSM Module
  sendSMS("AWD System Initialized");
}

void loop() {
  // Soil Moisture Reading
  soilMoistureValue = analogRead(soilMoisturePin);
  soilMoisturePercentage = map(soilMoistureValue, 0, 1023, 0, 100);
  
  // Water Depth Measurement
  waterDepth = sonar.ping_cm();
  
  // Debugging
  Serial.print("Soil Moisture (%): ");
  Serial.println(soilMoisturePercentage);
  Serial.print("Water Depth (cm): ");
  Serial.println(waterDepth);

  // Logic for Watering and Draining
  if (soilMoisturePercentage < minimumMoisture && waterDepth >= minimumWaterDepth) {
    // Turn on Water Pump (Flooding)
    digitalWrite(waterPumpPin, HIGH);
    sendSMS("Flooding Started.");
    Serial.println("Flooding Started.");
    lastFloodTime = millis();  // Start counting flooding time
  } else if (waterDepth > maximumWaterDepth) {
    // Turn on Drainage Pump (Draining)
    digitalWrite(drainagePumpPin, HIGH);
    sendSMS("Draining Started.");
    Serial.println("Draining Started.");
    lastDryTime = millis();    // Start counting drying time
  } else {
    // Turn off pumps
    digitalWrite(waterPumpPin, LOW);
    digitalWrite(drainagePumpPin, LOW);
  }

  // Time-based Flooding and Drying Phases (AWD)
  if (millis() - lastFloodTime >= floodingDuration) {
    // Switch to Drying Phase after Flooding Duration
    digitalWrite(waterPumpPin, LOW);  // Stop flooding
    digitalWrite(drainagePumpPin, HIGH);  // Start draining
    sendSMS("Flooding complete, switching to drying phase.");
    Serial.println("Flooding complete, switching to drying phase.");
  }
  
  if (millis() - lastDryTime >= dryingDuration) {
    // Switch to Wetting Phase after Drying Duration
    digitalWrite(drainagePumpPin, LOW);  // Stop draining
    digitalWrite(waterPumpPin, HIGH);  // Start flooding
    sendSMS("Drying complete, switching to flooding phase.");
    Serial.println("Drying complete, switching to flooding phase.");
  }

  // Power-saving sleep mode (no significant action, reduce power consumption)
  if (millis() - lastFloodTime < floodingDuration && millis() - lastDryTime < dryingDuration) {
    enterSleepMode();  // Enter low power mode when idle
  }

  delay(2000);  // Delay to stabilize the system
}

void sendSMS(const char* message) {
  gsmSerial.println("AT+CMGF=1"); // Set SMS format to text mode
  delay(100);
  gsmSerial.print("AT+CMGS=\"");
  gsmSerial.print(phone_number);
  gsmSerial.println("\"");
  delay(100);
  gsmSerial.println(message);
  delay(100);
  gsmSerial.write(26); // Send SMS
}

void enterSleepMode() {
  Serial.println("Entering Sleep Mode to Save Battery.");
  // Put the system to sleep for 'sleepTime' milliseconds
  delay(sleepTime);
  // Wake up from sleep and resume operation
}
