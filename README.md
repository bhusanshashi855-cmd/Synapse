# Synapse
Project 
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>

/* ---------- LCD + RTC ---------- */
LiquidCrystal_I2C lcd(0x27, 16, 2);
RTC_DS3231 rtc;

/* ---------- RFID ---------- */
#define SS_PIN 10
#define RST_PIN 9
MFRC522 rfid(SS_PIN, RST_PIN);

/* ---------- Relay Lock ---------- */
#define RELAY_PIN 7
bool boxLocked = true;

/* ---------- Auto-lock timer ---------- */
unsigned long unlockMillis = 0;
const unsigned long AUTO_LOCK_DELAY = 3000; // 3 seconds

/* ---------- Authorized UID ---------- */
byte allowedUID[4] = {0xBB, 0x72, 0x18, 0x02};

/* ---------- Clock Update ---------- */
unsigned long lastClockUpdate = 0;

/* ===================================================== */
bool checkUID() {
  for (byte i = 0; i < 4; i++)
    if (rfid.uid.uidByte[i] != allowedUID[i])
      return false;
  return true;
}

void lockBox() {
  digitalWrite(RELAY_PIN, HIGH); // Lock
  boxLocked = true;
  lcd.setCursor(0, 1);
  lcd.print("BOX: LOCKED  ");
}

void unlockBox() {
  digitalWrite(RELAY_PIN, LOW); // Unlock
  boxLocked = false;
  unlockMillis = millis(); // Start auto-lock countdown
  lcd.setCursor(0, 1);
  lcd.print("BOX: UNLOCKED");
}

void showClock() {
  DateTime now = rtc.now();
  lcd.setCursor(0, 0);
  lcd.print("Time:");
  if(now.hour()<10) lcd.print("0");
  lcd.print(now.hour()); lcd.print(":");
  if(now.minute()<10) lcd.print("0");
  lcd.print(now.minute()); lcd.print(":");
  if(now.second()<10) lcd.print("0");
  lcd.print(now.second());
}

/* ===================================================== */
void setup() {
  Serial.begin(9600);

  lcd.init(); 
  lcd.backlight();

  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, HIGH); // Lock by default

  /* RFID */
  SPI.begin();
  rfid.PCD_Init();

  /* RTC */
  if(!rtc.begin()){
    lcd.print("RTC ERROR"); 
    while(1); 
  }
  if(rtc.lostPower()) rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));

  lcd.setCursor(0,1);
  lcd.print("BOX: LOCKED  ");
}

/* ===================================================== */
void loop() {
  // Update clock every second
  if(millis() - lastClockUpdate > 1000){
    lastClockUpdate = millis();
    showClock();
  }

  // Auto-lock after 3 seconds
  if(!boxLocked && (millis() - unlockMillis >= AUTO_LOCK_DELAY)){
    lockBox();
  }

  // RFID scan
  if(rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()){
    Serial.print("UID: ");
    for(byte i=0; i<rfid.uid.size; i++){
      Serial.print(rfid.uid.uidByte[i], HEX); 
      Serial.print(" ");
    }
    Serial.println();

    if(checkUID()){
      unlockBox(); // Unlock on valid card scan
    } else {
      lcd.setCursor(0,1);
      lcd.print("ACCESS DENIED  ");
    }

    rfid.PICC_HaltA();
  }
}
