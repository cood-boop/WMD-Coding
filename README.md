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

// Variables
int soilMoistureValue;
int soilMoisturePercentage;
int waterDepth;
int minimumMoisture = 30;   // Minimum moisture percentage to trigger watering
int maximumWaterDepth = 15; // Max water depth in cm for AWD
int minimumWaterDepth = 5;  // Minimum water depth in cm for draining

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
  } else if (waterDepth > maximumWaterDepth) {
    // Turn on Drainage Pump
    digitalWrite(drainagePumpPin, HIGH);
    sendSMS("Draining Started.");
    Serial.println("Draining Started.");
  } else {
    // Turn off pumps
    digitalWrite(waterPumpPin, LOW);
    digitalWrite(drainagePumpPin, LOW);
  }

  delay(2000);  // Delay to allow the system to stabilize
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
