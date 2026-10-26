#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>
#include <Servo.h>

// LCD Configuration
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Servo
Servo doorServo;

// Password & Security Management
String correctPassword = "1234";
String enteredPassword = "";
int wrongAttempts = 0;

// Keypad Setup
const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};

// Keypad Pins
byte rowPins[ROWS] = {2, 3, 4, 5};
byte colPins[COLS] = {6, 7, 8, A1};

Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Pin Definitions
const int servoPin = 10;
const int trigPin = 11;
const int echoPin = 12;
const int pirPin = 13;
const int irPin = A0;
const int buzzerPin = A2;

// RGB LED Pins - Common Anode
const int redPin = A3;
const int greenPin = 9;

// Ultrasonic distance threshold for detecting visitors
const int visitorDistance = 5;   // Detects visitor within 5 cm

// Function to measure distance via Ultrasonic sensor
long getDistance() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);

  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH, 30000);

  if (duration == 0) {
    return 999;
  }

  long distance = duration * 0.034 / 2;
  return distance;
}

// RGB LED Control Functions (Common Anode)
void ledOff() {
  digitalWrite(redPin, HIGH);
  digitalWrite(greenPin, HIGH);
}

void redOn() {
  digitalWrite(redPin, LOW);
  digitalWrite(greenPin, HIGH);
}

void greenOn() {
  digitalWrite(redPin, HIGH);
  digitalWrite(greenPin, LOW);
}

void setup() {
  lcd.init();
  lcd.backlight();

  doorServo.attach(servoPin);
  doorServo.write(0); // Locked state

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(pirPin, INPUT);
  pinMode(irPin, INPUT);
  pinMode(buzzerPin, OUTPUT);

  pinMode(redPin, OUTPUT);
  pinMode(greenPin, OUTPUT);
  ledOff();

  lcd.setCursor(0, 0);
  lcd.print("Smart Door");
  lcd.setCursor(0, 1);
  lcd.print("System Ready");
  delay(1500);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("PIR Warming...");
  lcd.setCursor(0, 1);
  lcd.print("Please Wait");
  delay(30000); // PIR sensor warm-up time

  lcd.clear();
}

void loop() {
  long distance = getDistance();
  int motion = digitalRead(pirPin);
  int irValue = digitalRead(irPin);

  if (enteredPassword.length() == 0) {
    if (distance > 0 && distance <= visitorDistance) {
      lcd.setCursor(0, 0);
      lcd.print("Visitor Detected");
      lcd.setCursor(0, 1);
      lcd.print("Enter Password ");
    } 
    else if (motion == HIGH) {
      lcd.setCursor(0, 0);
      lcd.print("Motion Detected ");
      lcd.setCursor(0, 1);
      lcd.print("Enter Password ");
    } 
    else if (irValue == 0) {
      lcd.setCursor(0, 0);
      lcd.print("Door Closed     ");
      lcd.setCursor(0, 1);
      lcd.print("System Secure   ");
    }
    else {
      lcd.setCursor(0, 0);
      lcd.print("Door Open       ");
      lcd.setCursor(0, 1);
      lcd.print("Monitoring...   ");
    }
  }

  char key = keypad.getKey();

  if (key) {
    tone(buzzerPin, 1200, 80);

    if (key == '#') {
      checkPassword();
    }
    else if (key == '*') {
      enteredPassword = "";
      lcd.clear();
      lcd.print("Password Clear");
      delay(1000);
      lcd.clear();
    }
    else {
      enteredPassword += key;

      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Password:");
      lcd.setCursor(0, 1);

      for (int i = 0; i < enteredPassword.length(); i++) {
        lcd.print("*");
      }
    }
  }

  delay(200);
}

void checkPassword() {
  lcd.clear();

  if (enteredPassword == correctPassword) {
    wrongAttempts = 0;

    lcd.setCursor(0, 0);
    lcd.print("Access Granted");

    greenOn();

    tone(buzzerPin, 1000, 150);
    delay(200);
    tone(buzzerPin, 1500, 150);

    doorServo.write(90); // Unlock/Open door

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Door Opened");
    lcd.setCursor(0, 1);
    lcd.print("Wait 10 seconds");

    delay(10000); // Door stays open for 10 seconds

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Auto Locking...");
    doorServo.write(0); // Lock door again
    delay(1500);

    ledOff();
  }
  else {
    wrongAttempts++;

    lcd.setCursor(0, 0);
    lcd.print("Access Denied");
    lcd.setCursor(0, 1);
    lcd.print("Attempts: ");
    lcd.print(wrongAttempts);

    redOn();

    tone(buzzerPin, 400, 800);
    delay(1500);

    ledOff();

    if (wrongAttempts >= 3) {
      alarmMode();
      wrongAttempts = 0;
    }
  }

  enteredPassword = "";
  lcd.clear();
}

void alarmMode() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("SECURITY ALERT!");
  lcd.setCursor(0, 1);
  lcd.print("Too Many Tries");

  for (int i = 0; i < 6; i++) {
    redOn();
    tone(buzzerPin, 1000);
    delay(300);

    ledOff();
    noTone(buzzerPin);
    delay(300);
  }

  lcd.clear();
}