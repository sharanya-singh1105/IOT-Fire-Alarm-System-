# IoT Fire Alarm and Automatic Extinguishing System

This project is an IoT-based system designed to detect fire hazards and automatically trigger an extinguishing mechanism.

## Features
- Fire detection using sensors
- Automatic alert system
- Extinguishing response mechanism

## Technologies Used
- IoT
- Embedded Systems
- Basic Programming

## Description
This project helps in early fire detection and ensures safety through automation.

## CODE
#include <WiFi.h>
#include <FirebaseESP32.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// ********** WiFi Credentials **********
#define WIFI_SSID "**********"
#define WIFI_PASSWORD "********"

// ********** Firebase Credentials **********
#define API_KEY "AIzaSyDH4S9_rmOxMww725sWGLvibErl9hgWZXM"
#define DATABASE_URL "https://fire-alarm-f8d78-default-rtdb.asia-southeast1.firebasedatabase.app/"  // ✅ FIXED!

FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

// ********** Sensor & Actuator Pins **********
int flamePin = 18;     // Flame sensor output
int ledFlame = 5;      // LED if flame detected
int ledNoFlame = 19;   // LED if no flame
int buzzer = 23;       // Buzzer
int relayPin = 15;     // Relay module
int mq2Pin = 22;       // MQ2 gas sensor digital output

// ********** Temperature Sensor **********
#define ONE_WIRE_BUS 21
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(115200);

  pinMode(flamePin, INPUT);
  pinMode(ledFlame, OUTPUT);
  pinMode(ledNoFlame, OUTPUT);
  pinMode(buzzer, OUTPUT);
  pinMode(relayPin, OUTPUT);
  pinMode(mq2Pin, INPUT);

  sensors.begin();

  // ********** Connect WiFi **********
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(300);
  }
  Serial.println("\nWiFi Connected!");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());

  // ********** Firebase Setup **********
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;

  // Sign in anonymously or use token authentication
  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
  
  Serial.println("Firebase Initialized!");
  
  // Test connection
  if (Firebase.ready()) {
    Serial.println("✅ Firebase Connected Successfully!");
  } else {
    Serial.println("❌ Firebase Connection Failed!");
    Serial.println("Reason: " + fbdo.errorReason());
  }
}

void loop() {
  // ================= Flame Sensor =================
  int flameState = digitalRead(flamePin);
  int fireDetected = (flameState == LOW) ? 1 : 0;  // 1 = fire, 0 = no fire

  if (fireDetected) {
    digitalWrite(ledFlame, HIGH);
    digitalWrite(ledNoFlame, LOW);
    digitalWrite(buzzer, HIGH);
    digitalWrite(relayPin, HIGH);
    Serial.println("🔥 Flame Detected!");
  } else {
    digitalWrite(ledFlame, LOW);
    digitalWrite(ledNoFlame, HIGH);
    digitalWrite(buzzer, LOW);
    digitalWrite(relayPin, LOW);
    Serial.println("✅ No Flame");
  }

  // Update Firebase - matching your database structure
  if (Firebase.setInt(fbdo, "/FireStatus", fireDetected)) {
    Serial.println("✅ FireStatus updated");
  } else {
    Serial.println("❌ FireStatus failed: " + fbdo.errorReason());
  }

  if (Firebase.setInt(fbdo, "/FlameStatus", fireDetected)) {
    Serial.println("✅ FlameStatus updated");
  } else {
    Serial.println("❌ FlameStatus failed: " + fbdo.errorReason());
  }

  // ================= Temperature Sensor =================
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);

  Serial.print("🌡 Temperature: ");
  Serial.print(tempC);
  Serial.println(" °C");

  if (Firebase.setFloat(fbdo, "/Temperature", tempC)) {
    Serial.println("✅ Temperature updated");
  } else {
    Serial.println("❌ Temperature failed: " + fbdo.errorReason());
  }

  // ================= MQ2 Gas Sensor =================
  int gasState = digitalRead(mq2Pin);
  int gasDetected = (gasState == LOW) ? 1 : 0;  // 1 = gas detected, 0 = no gas

  if (gasDetected) {
    Serial.println("💨 Gas Detected!");
  } else {
    Serial.println("✅ No Gas");
  }

  if (Firebase.setInt(fbdo, "/GasLevel", gasDetected)) {
    Serial.println("✅ GasLevel updated");
  } else {
    Serial.println("❌ GasLevel failed: " + fbdo.errorReason());
  }

  // ================= Alert Status =================
  String alertStatus = "Safe";
  if (fireDetected || gasDetected || tempC > 50) {  // Adjust temperature threshold
    alertStatus = "Danger";
  }

  if (Firebase.setString(fbdo, "/Alert", alertStatus)) {
    Serial.println("✅ Alert updated: " + alertStatus);
  } else {
    Serial.println("❌ Alert failed: " + fbdo.errorReason());
  }

  // ================= Read Relay Status from Firebase =================
  if (Firebase.getInt(fbdo, "/RelayStatus")) {
    int relayStatus = fbdo.intData();
    digitalWrite(relayPin, relayStatus);
    Serial.println("📡 Relay status from Firebase: " + String(relayStatus));
  }

  Serial.println("----------------------------");
  delay(2000);  // Update every 2 seconds
}
