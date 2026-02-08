#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>

// ----- НАСТРОЙКИ -----
#define TOTAL_PLACES 3

int trigPins[TOTAL_PLACES] = {2, 4, 7};
int echoPins[TOTAL_PLACES] = {3, 5, 8};

#define SERVO_PIN 6

LiquidCrystal_I2C lcd(0x20, 16, 2); // если не работает — попробуй 0x27
Servo gate;

// ----- ПЕРЕМЕННЫЕ -----
long duration;
float distance[TOTAL_PLACES];
bool busy[TOTAL_PLACES];

void setup() {
  for (int i = 0; i < TOTAL_PLACES; i++) {
    pinMode(trigPins[i], OUTPUT);
    pinMode(echoPins[i], INPUT);
  }

  gate.attach(SERVO_PIN);
  gate.write(0);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("Parking system");
  lcd.setCursor(0, 1);
  lcd.print("3 places");
  delay(2000);
  lcd.clear();
}

void loop() {
  int freePlaces = 0;
  bool error = false;

  // ---- ОПРОС ДАТЧИКОВ ----
  for (int i = 0; i < TOTAL_PLACES; i++) {
    digitalWrite(trigPins[i], LOW);
    delayMicroseconds(2);
    digitalWrite(trigPins[i], HIGH);
    delayMicroseconds(10);
    digitalWrite(trigPins[i], LOW);

    duration = pulseIn(echoPins[i], HIGH, 30000);
    distance[i] = duration * 0.034 / 2;

    delay(60); // защита от помех

    // ---- ПРОВЕРКА ОШИБКИ ----
    if (distance[i] < 0.1 || distance[i] > 40 || duration == 0) {
      error = true;
    } 
    else if (distance[i] < 10) {
      busy[i] = true;
    } 
    else {
      busy[i] = false;
      freePlaces++;
    }
  }

  // ---- ERROR ----
  if (error) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("SENSOR ERROR");
    lcd.setCursor(0, 1);
    lcd.print("Check places");

    gate.write(0);
    delay(500);
    return;
  }

  // ---- LCD ВЫВОД ----
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Free: ");
  lcd.print(freePlaces);
  lcd.print("/3");

  lcd.setCursor(0, 1);
  for (int i = 0; i < TOTAL_PLACES; i++) {
    lcd.print("P");
    lcd.print(i + 1);
    lcd.print(busy[i] ? ":B " : ":F ");
  }

  // ---- ШЛАГБАУМ ----
  if (freePlaces > 0) {
    gate.write(90);
  } else {
    gate.write(0);
  }

  delay(500);
}
