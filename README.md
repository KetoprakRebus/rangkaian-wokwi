//lingkungan
#include <DHT.h>

// --- Pin Definitions ---
#define DHTPIN D1          // DHT22 Data pin
#define DHTTYPE DHT11      // Specify sensor type
DHT dht(DHTPIN, DHTTYPE);

#define FLAME_PIN D2       // Flame sensor digital output
#define BUZZER_PIN D5      // Buzzer positive pin
#define LED_RED D6         // Fire detected
#define LED_YELLOW D7      // High temperature warning
#define LED_GREEN D8       // Safe status

// --- Thresholds ---
const float TEMP_WARNING = 35.0; // Temperature in Celsius to trigger warning

void setup() {
  Serial.begin(115200);
  Serial.println("System Initializing...");
  
  dht.begin();
  
  // Set pin modes
  pinMode(FLAME_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  pinMode(LED_YELLOW, OUTPUT);
  pinMode(LED_GREEN, OUTPUT);
  
  // Start with all alarms off
  turnOffAlarms();
}

void loop() {
  // 1. Read the sensors
  // Note: Most flame sensors return HIGH when fire is detected (Active LOW)
  int flameState = digitalRead(FLAME_PIN); 
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();

  // Check if DHT read failed
  if (isnan(temperature) || isnan(humidity)) {
    Serial.println("Failed to read from DHT sensor!");
    delay(2000);
    return;
  }

  // Print data to Serial Monitor for debugging
  Serial.print("Temp: ");
  Serial.print(temperature);
  Serial.print("°C | Humidity: ");
  Serial.print(humidity);
  Serial.print("% | Flame State: ");
  Serial.println(flameState == HIGH ? "FIRE DETECTED!" : "Clear");

  // 2. Logic & Alerts
  if (flameState == HIGH) {
    // CRITICAL ALERT: Fire detected
    digitalWrite(LED_RED, HIGH);
    digitalWrite(LED_YELLOW, LOW);
    digitalWrite(LED_GREEN, LOW);
    
    // Pulse the buzzer
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
    
  } else if (temperature > TEMP_WARNING) {
    // WARNING ALERT: High temperature
    digitalWrite(LED_RED, LOW);
    digitalWrite(LED_YELLOW, HIGH);
    digitalWrite(LED_GREEN, LOW);
    digitalWrite(BUZZER_PIN, LOW); // Buzzer off
    delay(1000); // Wait a second before checking again
    
  } else {
    // NORMAL STATUS: All clear
    digitalWrite(LED_RED, LOW);
    digitalWrite(LED_YELLOW, LOW);
    digitalWrite(LED_GREEN, HIGH);
    digitalWrite(BUZZER_PIN, LOW);
    delay(2000); // Read sensor every 2 seconds
  }
}

// Helper function to ensure everything is off at startup
void turnOffAlarms() {
  digitalWrite(LED_RED, LOW);
  digitalWrite(LED_YELLOW, LOW);
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
}

//pertanian
#include "DHT.h"

// ======================
// PIN KOMPONEN
// ======================
#define SOIL_PIN 34       // Soil moisture AO ke D34 / GPIO34
#define DHT_PIN 4         // DATA DHT11/DHT22 ke GPIO4
#define RELAY_PIN 26      // IN relay ke GPIO26

// ======================
// TIPE DHT
// ======================
#define DHTTYPE DHT11
// Kalau pakai DHT22, ganti menjadi:
// #define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

// ======================
// RELAY
// ======================
// relay aktif LOW
#define RELAY_ON LOW
#define RELAY_OFF HIGH

// ======================
// BATAS SENSOR
// ======================
// Nilai soil ESP32: 0 - 4095
int batasTanahKering = 2500;
int batasTanahBasah = 1800;

// Batas suhu dan kelembapan udara
float batasSuhuPanas = 32.0;
float batasKelembapanUdaraRendah = 60.0;

bool pompaNyala = false;

void setup() {
  Serial.begin(115200);

  dht.begin();

  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, RELAY_OFF);

  Serial.println("Sistem Penyiraman Otomatis Tanaman Cabai");
  Serial.println("Soil Moisture + DHT11/DHT22 + Relay + Pompa");
}

void loop() {
  int nilaiTanah = analogRead(SOIL_PIN);

  float suhu = dht.readTemperature();
  float kelembapanUdara = dht.readHumidity();

  Serial.println("==================================");
  Serial.print("Nilai tanah: ");
  Serial.println(nilaiTanah);

  bool dhtTerbaca = true;

  if (isnan(suhu) || isnan(kelembapanUdara)) {
    Serial.println("DHT gagal terbaca");
    dhtTerbaca = false;
  } else {
    Serial.print("Suhu udara: ");
    Serial.print(suhu);
    Serial.println(" C");

    Serial.print("Kelembapan udara: ");
    Serial.print(kelembapanUdara);
    Serial.println(" %");
  }

  // ======================
  // KONDISI SENSOR
  // ======================

  bool tanahKering = nilaiTanah > batasTanahKering;
  bool tanahBasah = nilaiTanah < batasTanahBasah;

  bool udaraPanasDanKering = false;

  if (dhtTerbaca) {
    udaraPanasDanKering = suhu > batasSuhuPanas && kelembapanUdara < batasKelembapanUdaraRendah;
  }

  // ======================
  // LOGIKA POMPA
  // ======================

  if ((tanahKering || udaraPanasDanKering) && pompaNyala == false) {
    digitalWrite(RELAY_PIN, RELAY_ON);
    pompaNyala = true;

    Serial.println("Status: BUTUH PENYIRAMAN");
    Serial.println("Pompa: MENYALA");

    if (tanahKering) {
      Serial.println("Alasan: Tanah kering");
    }

    if (udaraPanasDanKering) {
      Serial.println("Alasan: Suhu panas dan udara kering");
    }
  }

  else if (tanahBasah && pompaNyala == true) {
    digitalWrite(RELAY_PIN, RELAY_OFF);
    pompaNyala = false;

    Serial.println("Status: TANAH SUDAH BASAH");
    Serial.println("Pompa: MATI");
  }

  else {
    Serial.println("Status: NORMAL");

    if (pompaNyala) {
      Serial.println("Pompa: MASIH MENYALA");
    } else {
      Serial.println("Pompa: MATI");
    }
  }

  delay(2000);
}

//peternakan
#define BLYNK_TEMPLATE_ID "TMPL6Nzswj4lM"
#define BLYNK_TEMPLATE_NAME "uap"
#define BLYNK_AUTH_TOKEN "YTeeyw9xnIBfDyZsch3jD_jBkJls9BHL"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>
#include <ESP32Servo.h>

char ssid[] = "heng";
char pass[] = "Hengky123";

#define DHTPIN 4
#define DHTTYPE DHT11
#define BUZZER_PIN 12
#define SERVO_PIN 13
#define TRIG_PIN 5
#define ECHO_PIN 18

DHT dht(DHTPIN, DHTTYPE);
Servo feederServo;
BlynkTimer timer;
bool sudahIsi = false;

void buzzerOn() {
  digitalWrite(BUZZER_PIN, HIGH);
}

void buzzerOff() {
  digitalWrite(BUZZER_PIN, LOW);
}

void kirimData()
{
  float suhu = dht.readTemperature();
  float hum = dht.readHumidity();

  if (!isnan(suhu) && !isnan(hum))
  {
    Blynk.virtualWrite(V0, suhu);
    Blynk.virtualWrite(V1, hum);

    if (suhu > 32)
      buzzerOn();
    else
      buzzerOff();
  }

  // Ultrasonik
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  float jarak = duration * 0.034 / 2;
  Blynk.virtualWrite(V2, jarak);

  if (jarak > 15 && !sudahIsi)
  {
    feederServo.write(90);
    delay(2000);
    feederServo.write(0);
    sudahIsi = true;
  }

  if (jarak <= 15)
    sudahIsi = false;
}

void setup()
{
  Serial.begin(115200);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  dht.begin();

  ESP32PWM::allocateTimer(0);
  ESP32PWM::allocateTimer(1);
  ESP32PWM::allocateTimer(2);
  ESP32PWM::allocateTimer(3);

  feederServo.attach(SERVO_PIN);
  feederServo.write(0);

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);
  timer.setInterval(3000L, kirimData);
}

void loop()
{
  Blynk.run();
  timer.run();
}

//rumah pintar
#include <Servo.h>

const int pinLDR = 4;
const int pinLED = 3;
const int pinHujan = 2;
const int pinServo = 9;

Servo servoJendela;

void setup() {
  Serial.begin(9600);
  
  pinMode(pinLDR, INPUT);
  pinMode(pinLED, OUTPUT);
  pinMode(pinHujan, INPUT);
  
  servoJendela.attach(pinServo);
  
  servoJendela.write(0); 
  Serial.println("Sistem Rumah Pintar (Digital LDR) Dimulai...");
}

void loop() {
  int statusGelap = digitalRead(pinLDR); 
  
  if (statusGelap == HIGH) {
    digitalWrite(pinLED, HIGH);
    Serial.print("Lampu: NYALA (GELAP)");
  } else {
    digitalWrite(pinLED, LOW);
    Serial.print("Lampu: MATI (TERANG)");
  }

  int statusHujan = digitalRead(pinHujan); 
  
  if (statusHujan == LOW) {
    servoJendela.write(90);
    Serial.println(" | Cuaca: HUJAN -> Atap MENUTUP (90°)");
  } else {
    servoJendela.write(0);
    Serial.println(" | Cuaca: CERAH -> Atap TERBUKA (0°)");
  }

  delay(1000); 
}
