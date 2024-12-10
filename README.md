#include <NewPing.h>
#include <BlynkSimpleEsp8266.h>

// Soil Moisture Sensor
const int soilMoisturePin = A0;  // Analog pin for soil moisture sensor
int soilMoistureValue;

// Ultrasonic Sensor
#define TRIG_PIN 9
#define ECHO_PIN 8
#define MAX_DISTANCE 200  // Max measurement distance in cm
NewPing sonar(TRIG_PIN, ECHO_PIN, MAX_DISTANCE);

// Floodgate Relay
const int floodgateRelayPin = 7;  // Digital pin for floodgate control

// Blynk Credentials
#define BLYNK_PRINT Serial
char auth[] = "YourBlynkAuthToken";
char ssid[] = "YourWiFiSSID";
char pass[] = "YourWiFiPassword";

// System Variables
int waterDepth;
int moisturePercentage;

// AWD Parameters
const int minimumMoisture = 30;  // Minimum soil moisture percentage to trigger watering
const int minimumDepth = 5;      // Minimum water depth (cm) to trigger floodgate

void setup() {
  // Serial Monitor for Debugging
  Serial.begin(9600);

  // Blynk Initialization
  Blynk.begin(auth, ssid, pass);

  // Pin Configurations
  pinMode(floodgateRelayPin, OUTPUT);  // Floodgate relay control
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
}

void loop() {
  Blynk.run();  // Keep Blynk running

  // Soil Moisture Sensor Reading
  soilMoistureValue = analogRead(soilMoisturePin);
  moisturePercentage = map(soilMoistureValue, 0, 1023, 0, 100);
  Serial.print("Soil Moisture (%): ");
  Serial.println(moisturePercentage);

  // Ultrasonic Sensor Water Depth Measurement
  waterDepth = sonar.ping_cm();  // Measure water depth in cm
  Serial.print("Water Depth (cm): ");
  Serial.println(waterDepth);

  // Logic for Automated Watering (AWD)
  if (moisturePercentage < minimumMoisture && waterDepth > minimumDepth) {
    digitalWrite(floodgateRelayPin, HIGH);  // Open floodgate
    Serial.println("Floodgate: OPEN");
  } else {
    digitalWrite(floodgateRelayPin, LOW);  // Close floodgate
    Serial.println("Floodgate: CLOSED");
  }

  // Send Data to Blynk App
  Blynk.virtualWrite(V1, moisturePercentage);  // Soil Moisture (%)
  Blynk.virtualWrite(V2, waterDepth);          // Water Depth (cm)

  delay(2000);  // Delay for stable readings
}
