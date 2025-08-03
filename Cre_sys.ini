//크레 냉난방 시스템
// 필수 라이브러리 포함
#include <ESP8266WiFi.h>              // WiFi 연결용
#include <WiFiUdp.h>                  // NTP 시간용 UDP
#include <NTPClient.h>                // NTP 클라이언트
#include <OneWire.h>                  // DS18B20 온도 센서용
#include <DallasTemperature.h>        // DS18B20 라이브러리
#include <LiquidCrystal_I2C.h>        // I2C 방식 LCD
#include <UniversalTelegramBot.h>     // Telegram 봇
#include <ESP8266HTTPClient.h>

// ---- WiFi 정보 입력 ----
const char* ssid = "Your_SSID";             // 와이파이 이름
const char* password = "Your_PASSWORD";     // 와이파이 비밀번호

// ---- Telegram 봇 정보 ----
#define BOTtoken "TELEGRAM_BOT_TOKEN"       // Telegram Bot Token
#define CHAT_ID "YOUR_CHAT_ID"              // 본인 Telegram ID

WiFiClientSecure client;  // HTTPS 연결용 클라이언트
UniversalTelegramBot bot(BOTtoken, client); // Telegram 봇 생성

// ---- 시간 설정 (서울 기준 +9시간) ----
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", 9 * 3600, 60000); // 60초마다 동기화

// ---- DS18B20 온도센서 설정 ----
#define ONE_WIRE_BUS D2              // 온도센서 데이터 핀
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

// ---- LCD 설정 ----
LiquidCrystal_I2C lcd(0x27, 20, 4);  // LCD 주소, 20글자 × 4줄

// ---- 릴레이 핀 설정 ----
#define FAN_RELAY_PIN D5            // 쿨링팬 릴레이 핀
#define HEATER_RELAY_PIN D6         // 전기장판 릴레이 핀

// ---- 현재 온도 및 설정 ----
float currentTemp = 0.0;             // 현재 측정된 온도
String currentSeason = "spring";    // 현재 계절 ("spring", "summer", "fall", "winter")
String currentTimePeriod = "day";   // 시간대 ("morning", "day", "evening", "night")

// ---- 계절별 온도 기준 ----
float heaterThreshold = 22.0;       // 전기장판 작동 기준 온도
float fanThreshold = 25.0;          // 쿨링팬 작동 기준 온도

// ---- 수동/자동 모드 ----
bool autoMode = true;               // 자동 모드 여부

// 초기 설정
void setup() {
  Serial.begin(115200);             // 시리얼 모니터 시작
  WiFi.begin(ssid, password);       // WiFi 연결 시작
  client.setInsecure();             // HTTPS 인증 무시

  // LCD 초기화
  lcd.begin();
  lcd.backlight();
  lcd.print("Connecting WiFi...");

  // 와이파이 연결 완료 대기
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  // NTP 클라이언트 시작
  timeClient.begin();

  // 온도센서 시작
  sensors.begin();

  // 릴레이 핀 출력 모드로 설정
  pinMode(FAN_RELAY_PIN, OUTPUT);
  pinMode(HEATER_RELAY_PIN, OUTPUT);

  digitalWrite(FAN_RELAY_PIN, LOW);     // 시작 시 꺼짐
  digitalWrite(HEATER_RELAY_PIN, LOW);

  lcd.clear();
  lcd.print("System Ready!");
  delay(2000);
}

// ---- 메인 루프 ----
void loop() {
  timeClient.update();              // 시간 업데이트
  sensors.requestTemperatures();    // 온도 측정 요청
  currentTemp = sensors.getTempCByIndex(0); // 온도 읽기

  updateTimePeriod();               // 시간대 계산
  updateSeason();                   // 계절 계산
  controlDevices();                 // 기기 제어
  printToLCD();                     // LCD 표시
  handleTelegram();                 // 텔레그램 명령 수신
  delay(5000);                      // 5초마다 반복
}

// ---- 시간대 계산 함수 ----
void updateTimePeriod() {
  int hour = timeClient.getHours();
  if (hour >= 6 && hour < 12) currentTimePeriod = "morning";
  else if (hour >= 12 && hour < 18) currentTimePeriod = "day";
  else if (hour >= 18 && hour < 22) currentTimePeriod = "evening";
  else currentTimePeriod = "night";
}

// ---- 계절에 따라 기준 온도 자동 변경 ----
void updateSeason() {
  int month = timeClient.getMonth();
  if (month >= 3 && month <= 5) {
    currentSeason = "spring";
    heaterThreshold = 22.0;
    fanThreshold = 25.0;
  } else if (month >= 6 && month <= 8) {
    currentSeason = "summer";
    heaterThreshold = -100.0;  // 여름엔 히터 끔
    fanThreshold = 26.0;
  } else if (month >= 9 && month <= 11) {
    currentSeason = "fall";
    heaterThreshold = 21.5;
    fanThreshold = 24.5;
  } else {
    currentSeason = "winter";
    heaterThreshold = 21.0;
    fanThreshold = 100.0;  // 겨울엔 팬 끔
  }
}

// ---- 온도에 따라 팬/히터 자동 제어 ----
void controlDevices() {
  if (!autoMode) return;  // 수동 모드면 아무것도 하지 않음

  // 팬 제어
  if (currentTemp > fanThreshold) {
    digitalWrite(FAN_RELAY_PIN, HIGH);
  } else {
    digitalWrite(FAN_RELAY_PIN, LOW);
  }

  // 히터 제어
  if (currentTemp < heaterThreshold) {
    digitalWrite(HEATER_RELAY_PIN, HIGH);
  } else {
    digitalWrite(HEATER_RELAY_PIN, LOW);
  }
}

// ---- LCD에 현재 정보 표시 ----
void printToLCD() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(currentTemp);
  lcd.print(" C");

  lcd.setCursor(0, 1);
  lcd.print("Season: ");
  lcd.print(currentSeason);

  lcd.setCursor(0, 2);
  lcd.print("Time: ");
  lcd.print(currentTimePeriod);

  lcd.setCursor(0, 3);
  lcd.print("Mode: ");
  lcd.print(autoMode ? "AUTO" : "MANUAL");
}

// ---- Telegram 명령어 처리 ----
void handleTelegram() {
  int numNewMessages = bot.getUpdates(bot.last_message_received + 1);

  while (numNewMessages) {
    for (int i = 0; i < numNewMessages; i++) {
      String text = bot.messages[i].text;
      String chat_id = bot.messages[i].chat_id;

      if (text == "/auto_mode") {
        autoMode = true;
        bot.sendMessage(chat_id, "자동 제어 모드로 전환됨", "");
      } else if (text == "/manual_mode") {
        autoMode = false;
        bot.sendMessage(chat_id, "수동 제어 모드로 전환됨", "");
      } else if (text == "/status") {
        String msg = "현재 온도: " + String(currentTemp) + " °C\n";
        msg += "계절: " + currentSeason + "\n";
        msg += "시간대: " + currentTimePeriod + "\n";
        msg += "팬 온도 기준: " + String(fanThreshold) + " °C\n";
        msg += "히터 온도 기준: " + String(heaterThreshold) + " °C\n";
        msg += "모드: " + String(autoMode ? "자동" : "수동");
        bot.sendMessage(chat_id, msg, "");
      }
    }
    numNewMessages = bot.getUpdates(bot.last_message_received + 1);
  }
}