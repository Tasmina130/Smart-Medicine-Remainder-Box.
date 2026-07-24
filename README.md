
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
RTC_DS3231 rtc;

// Pin definitions
const int buzzer = 8;
const int led = 9;
const int btnMode = 2; // Mode/Select
const int btnUp = 3; // Increase
const int btnReset = 4; // Save/Reset

// Alarm variables
int alarm1Hour = 8, alarm1Min = 0;
int alarm2Hour = 20, alarm2Min = 0;
bool alarm1Triggered = false, alarm2Triggered = false;
bool alarm1Active = false, alarm2Active = false; // New flags for active alarms
String activeAlarmMessage = ""; // Store which alarm is active

// Menu system
enum MenuState { MAIN, SET_ALARM1, SET_ALARM2 };
MenuState currentState = MAIN;
int editHour = 0, editMin = 0;
int cursorPos = 0; // 0=hours, 1=minutes
bool editing = false;

unsigned long lastButtonTime = 0;
const unsigned long debounceDelay = 250;

// For alarm buzzing pattern
unsigned long lastBuzzTime = 0;
bool buzzState = false;

void setup() {
lcd.init();
lcd.backlight();

if (!rtc.begin()) {
lcd.print("RTC Error!");
while (1);
}

// Set initial time (uncomment once)
// rtc.adjust(DateTime(2026, 7, 14, 10, 0, 0));

pinMode(buzzer, OUTPUT);
pinMode(led, OUTPUT);
pinMode(btnMode, INPUT_PULLUP);
pinMode(btnUp, INPUT_PULLUP);
pinMode(btnReset, INPUT_PULLUP);

digitalWrite(buzzer, LOW);
digitalWrite(led, LOW);

lcd.clear();
lcd.print("Medicine Box");
lcd.setCursor(0, 1);
lcd.print("v1.0");
delay(1500);
lcd.clear();
}

void loop() {
DateTime now = rtc.now();

handleButtons(now);
checkAlarms(now);

// Handle active alarm buzzing and display
if (alarm1Active || alarm2Active) {
handleAlarmBuzzing();
displayAlarmActive();
} else {
updateDisplay(now);
}

delay(150);
}

void handleAlarmBuzzing() {
unsigned long currentMillis = millis();
if (currentMillis - lastBuzzTime > 300) { // Blink every 300ms
lastBuzzTime = currentMillis;
buzzState = !buzzState;
digitalWrite(buzzer, buzzState);
digitalWrite(led, buzzState);
}
}

void displayAlarmActive() {
lcd.clear();
lcd.setCursor(0, 0);
lcd.print("*** REMINDER ***");
lcd.setCursor(0, 1);
lcd.print(activeAlarmMessage);
}

void handleButtons(DateTime now) {
bool modePress = digitalRead(btnMode) == LOW;
bool upPress = digitalRead(btnUp) == LOW;
bool resetPress = digitalRead(btnReset) == LOW;

if (modePress && millis() - lastButtonTime > debounceDelay) {
lastButtonTime = millis();
handleMode(now);
}

if (upPress && millis() - lastButtonTime > debounceDelay) {
lastButtonTime = millis();
handleUp(now);
}

if (resetPress && millis() - lastButtonTime > debounceDelay) {
lastButtonTime = millis();
handleReset(now); // Pass 'now' to the function
}
}

void handleMode(DateTime now) {
// If alarm is active, stop it first
if (alarm1Active || alarm2Active) {
stopAlarm("Medicine has taken");
return;
}

if (currentState == MAIN) {
// Enter alarm setting mode
currentState = SET_ALARM1;
editHour = alarm1Hour;
editMin = alarm1Min;
cursorPos = 0;
editing = false;
lcd.clear();
lcd.print("Alarm 1");
}
else if (currentState == SET_ALARM1) {
if (!editing) {
editing = true;
cursorPos = 0;
} else {
cursorPos = (cursorPos == 0) ? 1 : 0;
}
}
else if (currentState == SET_ALARM2) {
if (!editing) {
editing = true;
cursorPos = 0;
} else {
cursorPos = (cursorPos == 0) ? 1 : 0;
}
}
}

void handleUp(DateTime now) {
if (!editing) {
return;
}

if (cursorPos == 0) {
editHour++;
if (editHour > 23) editHour = 0;
} else {
editMin++;
if (editMin > 59) editMin = 0;
}
}

void handleReset(DateTime now) {
// If alarm is active, stop it with Reset button
if (alarm1Active || alarm2Active) {
stopAlarm("Medicine has taken");
return;
}

if (currentState == SET_ALARM1) {
alarm1Hour = editHour;
alarm1Min = editMin;

currentState = SET_ALARM2;  
editHour = alarm2Hour;  
editMin = alarm2Min;  
cursorPos = 0;  
editing = false;  
lcd.clear();  
lcd.print("Alarm 2");  
  
digitalWrite(led, HIGH);  
delay(200);  
digitalWrite(led, LOW);
}
else if (currentState == SET_ALARM2) {
alarm2Hour = editHour;
alarm2Min = editMin;

currentState = MAIN;  
editing = false;  
lcd.clear();  
  
digitalWrite(led, HIGH);  
delay(200);  
digitalWrite(led, LOW);  
delay(100);  
digitalWrite(led, HIGH);  
delay(200);  
digitalWrite(led, LOW);  
  
lcd.setCursor(0, 0);  
lcd.print("Alarms Saved!");  
lcd.setCursor(0, 1);  
lcd.print("A1:");  
printDigits(alarm1Hour);  
lcd.print(":");  
printDigits(alarm1Min);  
lcd.print(" A2:");  
printDigits(alarm2Hour);  
lcd.print(":");  
printDigits(alarm2Min);  
delay(1500);  
lcd.clear();
}
else if (currentState == MAIN) {
// This part is now handled by the alarm check at the beginning
}
}

void stopAlarm(String message) {
// Stop buzzer and LED
digitalWrite(buzzer, LOW);
digitalWrite(led, LOW);
alarm1Active = false;
alarm2Active = false;
alarm1Triggered = false;
alarm2Triggered = false;
buzzState = false;
activeAlarmMessage = "";

// Show confirmation message for 5 seconds
lcd.clear();
lcd.setCursor(0, 0);
lcd.print("*** REMINDER ***");
lcd.setCursor(0, 1);
lcd.print(message);
delay(5000); // Increased to 5 seconds
lcd.clear();
}

void updateDisplay(DateTime now) {
switch(currentState) {
case MAIN:
displayMain(now);
break;
case SET_ALARM1:
displaySetAlarm("1", editHour, editMin);
break;
case SET_ALARM2:
displaySetAlarm("2", editHour, editMin);
break;
}
}

void displayMain(DateTime now) {
lcd.setCursor(0, 0);
lcd.print("Time: ");
printDigits(now.hour());
lcd.print(":");
printDigits(now.minute());
lcd.print(":");
printDigits(now.second());

lcd.setCursor(0, 1);
lcd.print("A1:");
printDigits(alarm1Hour);
lcd.print(":");
printDigits(alarm1Min);
lcd.print(" A2:");
printDigits(alarm2Hour);
lcd.print(":");
printDigits(alarm2Min);

if (alarm1Active || alarm2Active) {
lcd.setCursor(15, 0);
lcd.print("!");
}
}

void displaySetAlarm(String num, int hour, int min) {
lcd.setCursor(0, 0);
lcd.print("Alarm ");
lcd.print(num);

if (editing) {
lcd.print(" EDIT");
lcd.setCursor(12, 0);
lcd.print("UP+");
} else {
lcd.print(" SAVE");
lcd.setCursor(12, 0);
lcd.print("MODE");
}

lcd.setCursor(0, 1);

if (editing && cursorPos == 0) {
lcd.print(">");
} else {
lcd.print(" ");
}
lcd.print("H:");
printDigits(hour);

lcd.setCursor(8, 1);
if (editing && cursorPos == 1) {
lcd.print(">");
} else {
lcd.print(" ");
}
lcd.print("M:");
printDigits(min);

lcd.setCursor(14, 1);
if (editing) {
lcd.print("SV");
} else {
lcd.print(" ");
}
}

void checkAlarms(DateTime now) {
int h = now.hour();
int m = now.minute();
int s = now.second();

// Alarm 1
if (!alarm1Triggered && !alarm1Active && h == alarm1Hour && m == alarm1Min && s == 0) {
triggerAlarm("Take Medicine 1!");
alarm1Triggered = true;
alarm1Active = true;
activeAlarmMessage = "Take Medicine 1!";
}

// Alarm 2
if (!alarm2Triggered && !alarm2Active && h == alarm2Hour && m == alarm2Min && s == 0) {
triggerAlarm("Take Medicine 2!");
alarm2Triggered = true;
alarm2Active = true;
activeAlarmMessage = "Take Medicine 2!";
}
}

void triggerAlarm(String message) {
lcd.clear();
lcd.setCursor(0, 0);
lcd.print("*** REMINDER ***");
lcd.setCursor(0, 1);
lcd.print(message);

// Start the alarm buzzing - now handled in loop
lastBuzzTime = millis();
buzzState = false;
}

void printDigits(int num) {
if (num < 10) lcd.print("0");
lcd.print(num);
}
