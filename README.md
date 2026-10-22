#include <Arduino.h>
#include <Wire.h>
#include <SPI.h>
#include <U8g2lib.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_LIS3DH.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>

const char* ssid     = "wifi_name";
const char* password = "wifi_password";

#define BOT_TOKEN "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
#define CHAT_ID   "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);

#define OLED_CS   15
#define OLED_DC   2
#define OLED_RES  4
#define MOTOR     27
#define IR_SENSOR_1   13
#define IR_SENSOR_2   33
#define BUZZER_PIN   16
#define START_PIN    14

#define LIS3DH_ADDR_1 0x18
#define LIS3DH_ADDR_2 0x19

typedef struct { char text[128]; } TelegramMessage;
QueueHandle_t telegramQueue;
TaskHandle_t TelegramTaskHandle;

volatile bool flag_siren = false;
volatile int motor_intensity = 255;
volatile bool show_custom_msg = false;
volatile char custom_msg_text[45] = "";
volatile unsigned long custom_msg_end = 0;

U8G2_SH1106_128X64_NONAME_1_4W_HW_SPI u8g2(U8G2_R0, OLED_CS, OLED_DC, OLED_RES);
Adafruit_LIS3DH lis;

const int eyeSize = 35;
unsigned long lastBlinkTime = 0;
bool isBlinking = false;
unsigned long blinkEndTime = 0;
int blinkInterval = 3000;

bool showMessage = false;
String currentMessage = "";
unsigned long messageEndTime = 0;

float offsetX = 0, offsetY = 0, offsetZ = 0;
bool calibrated = false;
bool lis3dh_found = false;

unsigned long lastIRTime = 0;
unsigned long lastMovementTime = 0;
const unsigned long EVENT_COOLDOWN = 3000;
const unsigned long MOVEMENT_COOLDOWN = 1500;
const unsigned long DISPLAY_DURATION = 1500;

unsigned long lastSirenToggle = 0;
bool sirenHighPitch = true;

int activeToneSequence = 0;
int toneStep = 0;
unsigned long nextToneTime = 0;

int activeVibSequence = 0;
int vibStep = 0;
unsigned long nextVibTime = 0;

unsigned long lastFrameTime = 0;
const int FRAME_INTERVAL = 30;

void setMotor(int state);
void setMotorIntensity(int duty);
void queueTelegramMessage(const char* text);
void startVibration(int pattern);
void startTone(int toneId);
void handleNonBlockingEffects();
void drawDisplay();
bool initLIS3DH();

void setup() {
  Serial.begin(115200);
  Serial.println("\n=== DESKGUARDIAN V6 (SINGLE MOTOR) STARTING ===\n");
  
  ledcSetup(0, 5000, 8);
  ledcAttachPin(MOTOR, 0);
  setMotorIntensity(0);
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(IR_SENSOR_1, INPUT_PULLUP);
  pinMode(IR_SENSOR_2, INPUT_PULLUP);
  pinMode(START_PIN, INPUT_PULLUP);
  digitalWrite(BUZZER_PIN, LOW);
  
  Wire.begin(21, 22);
  Wire.setClock(100000);
  delay(100);
  
  u8g2.begin();
  u8g2.clearDisplay();
  
  u8g2.firstPage();
  do {
    u8g2.setFont(u8g2_font_helvB12_tf);
    u8g2.drawStr(15, 30, "Press Start...");
  } while (u8g2.nextPage());
  
  Serial.println("Waiting for start button on GPIO 14...");
  while (digitalRead(START_PIN) == HIGH) {
    delay(50);
  }
  
  Serial.println("Button pressed!");
  tone(BUZZER_PIN, 880, 100);
  delay(100);
  noTone(BUZZER_PIN);
  
  u8g2.firstPage();
  do {
    u8g2.setFont(u8g2_font_helvB12_tf);
    u8g2.drawStr(20, 30, "Initializing...");
  } while (u8g2.nextPage());
  
  if (!initLIS3DH()) {
    Serial.println("WARNING: LIS3DH not found! Movement detection disabled.");
    u8g2.firstPage();
    do {
      u8g2.setFont(u8g2_font_helvB08_tf);
      u8g2.drawStr(10, 30, "LIS3DH NOT FOUND");
      u8g2.drawStr(10, 45, "Movement DISABLED");
    } while (u8g2.nextPage());
    delay(2000);
  } else {
    Serial.println("LIS3DH found and calibrated!");
    
    u8g2.firstPage();
    do {
      u8g2.setFont(u8g2_font_helvB14_tf);
      int w1 = u8g2.getStrWidth("Desk");
      u8g2.drawStr((128 - w1)/2, 30, "Desk");
      int w2 = u8g2.getStrWidth("Guardian");
      u8g2.drawStr((128 - w2)/2, 55, "Guardian");
    } while (u8g2.nextPage());
    
    delay(3000);
    
    Serial.println("Calibrating LIS3DH...");
    float sumX=0, sumY=0, sumZ=0;
    for(int i=0; i<100; i++) {
      sensors_event_t event; 
      if (lis.getEvent(&event)) {
        sumX += event.acceleration.x; 
        sumY += event.acceleration.y; 
        sumZ += event.acceleration.z;
      }
      delay(10);
    }
    offsetX = sumX/100.0; 
    offsetY = sumY/100.0; 
    offsetZ = (sumZ/100.0) - 9.81;
    
    Serial.print("Offsets - X:"); Serial.print(offsetX);
    Serial.print(" Y:"); Serial.print(offsetY);
    Serial.print(" Z:"); Serial.println(offsetZ);
    
    calibrated = true;
  }
  
  u8g2.clearDisplay();
  u8g2.firstPage();
  do {
    u8g2.setFont(u8g2_font_helvB10_tf);
    int w1 = u8g2.getStrWidth("Keep on desk");
    u8g2.drawStr((128 - w1)/2, 25, "Keep on desk");
    int w2 = u8g2.getStrWidth("I'll take care");
    u8g2.drawStr((128 - w2)/2, 50, "I'll take care");
  } while (u8g2.nextPage());
  
  delay(1000);
  setMotorIntensity(80);
  delay(80);
  setMotorIntensity(0);
  
  tone(BUZZER_PIN, 523, 100);
  delay(100);
  tone(BUZZER_PIN, 659, 150);
  delay(150);
  tone(BUZZER_PIN, 784, 200);
  delay(200);
  noTone(BUZZER_PIN);
  
  u8g2.clearDisplay();
  
  telegramQueue = xQueueCreate(10, sizeof(TelegramMessage));
  xTaskCreatePinnedToCore(networkTask, "NetworkTask", 8192, NULL, 1, &TelegramTaskHandle, 0);
  
  lastBlinkTime = millis();
  blinkInterval = random(2000, 5000);
  
  queueTelegramMessage("🔘 DeskGuardian started by button press");
  delay(100);
  queueTelegramMessage("✅ DeskGuardian V6 is ONLINE! (Single Motor)");
  
  Serial.println("Setup complete! Entering main loop...\n");
}

bool initLIS3DH() {
  Serial.println("Searching for LIS3DH...");
  
  byte error, address;
  int nDevices = 0;
  for(address = 1; address < 127; address++) {
    Wire.beginTransmission(address);
    error = Wire.endTransmission();
    if (error == 0) {
      Serial.print("I2C device found at 0x");
      if (address < 16) Serial.print("0");
      Serial.println(address, HEX);
      nDevices++;
    }
  }
  
  if (nDevices == 0) {
    Serial.println("No I2C devices found! Check wiring.");
    return false;
  }
  
  Serial.print("Trying LIS3DH at 0x18... ");
  if (lis.begin(LIS3DH_ADDR_1)) {
    Serial.println("SUCCESS!");
    lis3dh_found = true;
    lis.setRange(LIS3DH_RANGE_4_G);
    lis.setDataRate(LIS3DH_DATARATE_50_HZ);
    return true;
  }
  Serial.println("FAILED");
  
  Serial.print("Trying LIS3DH at 0x19... ");
  if (lis.begin(LIS3DH_ADDR_2)) {
    Serial.println("SUCCESS!");
    lis3dh_found = true;
    lis.setRange(LIS3DH_RANGE_4_G);
    lis.setDataRate(LIS3DH_DATARATE_50_HZ);
    return true;
  }
  Serial.println("FAILED");
  
  return false;
}

void setMotor(int state) {
  if (state) {
    ledcWrite(0, motor_intensity);
  } else {
    ledcWrite(0, 0);
  }
}

void setMotorIntensity(int duty) {
  ledcWrite(0, constrain(duty, 0, 255));
}

void networkTask(void *pvParameters) {
  u8g2.firstPage();
  do {
    u8g2.setFont(u8g2_font_helvB10_tf);
    u8g2.drawStr(10, 30, "WiFi connecting...");
  } while (u8g2.nextPage());
  
  WiFi.begin(ssid, password);
  int timeout = 0;
  while (WiFi.status() != WL_CONNECTED && timeout < 40) {
    vTaskDelay(pdMS_TO_TICKS(500));
    timeout++;
    Serial.print(".");
  }
  Serial.println();
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("WiFi connected! IP: ");
    Serial.println(WiFi.localIP());
    u8g2.firstPage();
    do {
      u8g2.setFont(u8g2_font_helvB10_tf);
      u8g2.drawStr(10, 30, "WiFi OK!");
    } while (u8g2.nextPage());
    vTaskDelay(pdMS_TO_TICKS(800));
    u8g2.clearDisplay();
    
    secured_client.setInsecure();
    bot.sendMessage(CHAT_ID, "✅ DeskGuardian V6 is ONLINE! (Single Motor)", "");
  } else {
    Serial.println("WiFi connection FAILED!");
    u8g2.firstPage();
    do {
      u8g2.setFont(u8g2_font_helvB10_tf);
      u8g2.drawStr(10, 30, "WiFi FAIL");
    } while (u8g2.nextPage());
    vTaskDelay(pdMS_TO_TICKS(800));
    u8g2.clearDisplay();
  }
  
  unsigned long lastPoll = 0;
  TelegramMessage msg;

  for(;;) {
    if (xQueueReceive(telegramQueue, &msg, 0) == pdTRUE) {
      if (WiFi.status() == WL_CONNECTED) {
        bot.sendMessage(CHAT_ID, msg.text, "");
      }
    }
    
    if (WiFi.status() == WL_CONNECTED && (millis() - lastPoll > 2000)) {
      int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
      for (int i = 0; i < numNewMessages; i++) {
        String text = bot.messages[i].text;
        String chat_id = String(bot.messages[i].chat_id);
        
        if (text == "/status") {
          String status = "🟢 DeskGuardian ONLINE\n";
          status += calibrated ? "✅ Movement ACTIVE\n" : "⚠️ Movement DISABLED\n";
          status += "IR sensors ACTIVE\n";
          status += flag_siren ? "🚨 SIREN ON" : "🔇 Siren OFF";
          bot.sendMessage(chat_id, status, "");
        }
        else if (text == "/ping") {
          bot.sendMessage(chat_id, "🏓 Pong! DeskGuardian is alive", "");
        }
        else if (text == "/siren") {
          flag_siren = !flag_siren; 
          bot.sendMessage(chat_id, flag_siren ? "🚨 Police Siren ACTIVATED!" : "🛑 Siren DEACTIVATED", "");
        }
        else if (text == "/stopcommand") {
          flag_siren = false;
          activeToneSequence = 0;
          activeVibSequence = 0;
          noTone(BUZZER_PIN);
          setMotorIntensity(0);
          bot.sendMessage(chat_id, "🛑 All effects STOPPED", "");
        }
        else if (text.startsWith("/msg ")) {
          String custom = text.substring(5);
          if (custom.length() > 40) {
            bot.sendMessage(chat_id, "⚠️ Max 40 characters", "");
          } else {
            strcpy((char*)custom_msg_text, custom.c_str());
            show_custom_msg = true;
            custom_msg_end = millis() + 3000;
            bot.sendMessage(chat_id, "💬 Displayed: " + custom, "");
          }
        }
        else if (text == "/goaway") {
          show_custom_msg = false;
          currentMessage = "GO AWAY!!";
          showMessage = true;
          messageEndTime = millis() + DISPLAY_DURATION;
          startVibration(3);
          startTone(2);
          bot.sendMessage(chat_id, "✅ GO AWAY!! displayed", "");
        }
        else if (text == "/donttouch") {
          show_custom_msg = false;
          currentMessage = "DON'T TOUCH";
          showMessage = true;
          messageEndTime = millis() + DISPLAY_DURATION;
          startVibration(2);
          startTone(1);
          bot.sendMessage(chat_id, "✅ DON'T TOUCH displayed", "");
        }
        else if (text == "/leave") {
          show_custom_msg = false;
          currentMessage = "LEAVE NOW!!";
          showMessage = true;
          messageEndTime = millis() + DISPLAY_DURATION;
          startVibration(3);
          startTone(2);
          bot.sendMessage(chat_id, "✅ LEAVE NOW!! displayed", "");
        }
        else if (text == "/backup") {
          show_custom_msg = false;
          currentMessage = "BACK OFF!!";
          showMessage = true;
          messageEndTime = millis() + DISPLAY_DURATION;
          startVibration(3);
          startTone(4);
          bot.sendMessage(chat_id, "✅ BACK OFF!! displayed", "");
        }
      }
      lastPoll = millis();
    }
    vTaskDelay(pdMS_TO_TICKS(50));
  }
}

void loop() {
  unsigned long currentMillis = millis();
  
  bool ir1 = (digitalRead(IR_SENSOR_1) == LOW);
  bool ir2 = (digitalRead(IR_SENSOR_2) == LOW);
  
  if ((ir1 || ir2) && (currentMillis - lastIRTime > EVENT_COOLDOWN)) {
    if (ir1 && ir2) {
      currentMessage = "GO AWAY!!";
      showMessage = true;
      messageEndTime = currentMillis + DISPLAY_DURATION;
      startVibration(3);
      startTone(2);
      queueTelegramMessage("⚠️⚠️ INTRUDER! GO AWAY!!");
    } else {
      currentMessage = "DON'T TOUCH";
      showMessage = true;
      messageEndTime = currentMillis + DISPLAY_DURATION;
      startVibration(2);
      startTone(1);
      queueTelegramMessage("🚫 Someone touched me!");
    }
    lastIRTime = currentMillis;
  }

  if (calibrated && lis3dh_found && (currentMillis - lastMovementTime > MOVEMENT_COOLDOWN)) {
    sensors_event_t event; 
    if (lis.getEvent(&event)) {
      float x = event.acceleration.x - offsetX;
      float y = event.acceleration.y - offsetY;
      float z = event.acceleration.z - offsetZ;
      float mag = sqrt(x*x + y*y + z*z);
      
      if (mag > 11.0 || mag < 8.5) {
        currentMessage = "DOWN!!";
        showMessage = true;
        messageEndTime = currentMillis + DISPLAY_DURATION;
        startVibration(3);
        startTone(4);
        queueTelegramMessage("💥 DESK SHAKE DETECTED!");
        lastMovementTime = currentMillis;
      } else if (mag > 10.1 || mag < 9.5) {
        currentMessage = "WHO IS THIS?";
        showMessage = true;
        messageEndTime = currentMillis + DISPLAY_DURATION;
        startVibration(2);
        startTone(3);
        queueTelegramMessage("👀 Suspicious movement detected!");
        lastMovementTime = currentMillis;
      }
    }
  }

  if (!isBlinking && (currentMillis - lastBlinkTime > blinkInterval)) {
    isBlinking = true;
    blinkEndTime = currentMillis + 150;
    lastBlinkTime = currentMillis;
    blinkInterval = random(2000, 5000);
  }
  if (isBlinking && currentMillis > blinkEndTime) {
    isBlinking = false;
  }

  if (showMessage && currentMillis > messageEndTime) {
    showMessage = false;
  }
  if (show_custom_msg && currentMillis > custom_msg_end) {
    show_custom_msg = false;
  }

  handleNonBlockingEffects();

  drawDisplay();
  
  delay(FRAME_INTERVAL);
}

void drawDisplay() {
  u8g2.firstPage();
  do {
    if (show_custom_msg) {
      u8g2.setFont(u8g2_font_helvB10_tf);
      String fullMsg = String((char*)custom_msg_text);
      if (fullMsg.length() <= 20) {
        u8g2.setFont(u8g2_font_helvB12_tf);
        u8g2.drawStr((128 - u8g2.getStrWidth(fullMsg.c_str()))/2, 45, fullMsg.c_str());
      } else {
        u8g2.setFont(u8g2_font_helvB10_tf);
        String line1 = fullMsg.substring(0, 20);
        String line2 = fullMsg.substring(20, 40);
        u8g2.drawStr((128 - u8g2.getStrWidth(line1.c_str()))/2, 35, line1.c_str());
        if (line2.length() > 0) {
          u8g2.drawStr((128 - u8g2.getStrWidth(line2.c_str()))/2, 50, line2.c_str());
        }
      }
    }
    else if (showMessage) {
      if (currentMessage == "GO AWAY!!" || currentMessage == "DOWN!!") {
        u8g2.setFont(u8g2_font_helvB18_tf);
      } else {
        u8g2.setFont(u8g2_font_helvB14_tf);
      }
      int w = u8g2.getStrWidth(currentMessage.c_str());
      u8g2.drawStr((128 - w)/2, 45, currentMessage.c_str());
    }
    else {
      int half = eyeSize/2;
      if (isBlinking) {
        u8g2.drawLine(40 - half, 32, 40 + half, 32);
        u8g2.drawLine(88 - half, 32, 88 + half, 32);
      } else {
        u8g2.drawBox(40 - half, 32 - half, eyeSize, eyeSize);
        u8g2.drawBox(88 - half, 32 - half, eyeSize, eyeSize);
      }
    }
  } while(u8g2.nextPage());
}

void startVibration(int pattern) {
  activeVibSequence = pattern;
  vibStep = 0;
  nextVibTime = millis();
}

void startTone(int toneId) {
  if (flag_siren) return;
  activeToneSequence = toneId;
  toneStep = 0;
  nextToneTime = millis();
}

void queueTelegramMessage(const char* text) {
  TelegramMessage msg;
  strlcpy(msg.text, text, sizeof(msg.text));
  xQueueSend(telegramQueue, &msg, 0);
}

void handleNonBlockingEffects() {
  unsigned long t = millis();

  if (flag_siren) {
    if (t - lastSirenToggle > 200) {
      sirenHighPitch = !sirenHighPitch;
      tone(BUZZER_PIN, sirenHighPitch ? 1000 : 600);
      setMotorIntensity(sirenHighPitch ? motor_intensity : motor_intensity/2);
      lastSirenToggle = t;
    }
    return;
  } else if (!flag_siren && lastSirenToggle > 0) {
    noTone(BUZZER_PIN);
    setMotorIntensity(0);
    lastSirenToggle = 0;
  }

  if (activeToneSequence > 0 && t >= nextToneTime) {
    switch(activeToneSequence) {
      case 1:
        if (toneStep == 0) { tone(BUZZER_PIN, 880); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 1) { tone(BUZZER_PIN, 660); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 2) { tone(BUZZER_PIN, 880); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 3) { tone(BUZZER_PIN, 660); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 4) { tone(BUZZER_PIN, 880); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 5) { tone(BUZZER_PIN, 660); nextToneTime = t + 80; toneStep++; }
        else { noTone(BUZZER_PIN); activeToneSequence = 0; }
        break;
        
      case 2:
        if (toneStep == 0) { tone(BUZZER_PIN, 1200); nextToneTime = t + 150; toneStep++; }
        else if (toneStep == 1) { tone(BUZZER_PIN, 800); nextToneTime = t + 200; toneStep++; }
        else if (toneStep == 2) { tone(BUZZER_PIN, 600); nextToneTime = t + 100; toneStep++; }
        else if (toneStep == 3) { tone(BUZZER_PIN, 400); nextToneTime = t + 200; toneStep++; }
        else { noTone(BUZZER_PIN); activeToneSequence = 0; }
        break;
        
      case 3:
        if (toneStep == 0) { tone(BUZZER_PIN, 523); nextToneTime = t + 100; toneStep++; }
        else if (toneStep == 1) { tone(BUZZER_PIN, 587); nextToneTime = t + 100; toneStep++; }
        else if (toneStep == 2) { tone(BUZZER_PIN, 523); nextToneTime = t + 100; toneStep++; }
        else if (toneStep == 3) { tone(BUZZER_PIN, 494); nextToneTime = t + 150; toneStep++; }
        else if (toneStep == 4) { tone(BUZZER_PIN, 440); nextToneTime = t + 200; toneStep++; }
        else { noTone(BUZZER_PIN); activeToneSequence = 0; }
        break;
        
      case 4:
        if (toneStep == 0) { tone(BUZZER_PIN, 330); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 1) { tone(BUZZER_PIN, 392); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 2) { tone(BUZZER_PIN, 523); nextToneTime = t + 80; toneStep++; }
        else if (toneStep == 3) { tone(BUZZER_PIN, 659); nextToneTime = t + 150; toneStep++; }
        else if (toneStep == 4) { tone(BUZZER_PIN, 784); nextToneTime = t + 200; toneStep++; }
        else { noTone(BUZZER_PIN); activeToneSequence = 0; }
        break;
    }
  }

  if (activeVibSequence > 0 && t >= nextVibTime) {
    if (activeVibSequence == 2) {
      if (vibStep == 0) { 
        setMotorIntensity(motor_intensity); 
        nextVibTime = t + 120; 
        vibStep++; 
      }
      else { 
        setMotorIntensity(0); 
        activeVibSequence = 0; 
      }
    }
    else if (activeVibSequence == 3) {
      if (vibStep == 0) { 
        setMotorIntensity(motor_intensity); 
        nextVibTime = t + 80; 
        vibStep++; 
      }
      else if (vibStep == 1) { 
        setMotorIntensity(0); 
        nextVibTime = t + 80; 
        vibStep++; 
      }
      else if (vibStep == 2) { 
        setMotorIntensity(motor_intensity); 
        nextVibTime = t + 80; 
        vibStep++; 
      }
      else { 
        setMotorIntensity(0); 
        activeVibSequence = 0; 
      }
    }
  }
}
