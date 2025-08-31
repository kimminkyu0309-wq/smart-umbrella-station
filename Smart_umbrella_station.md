# smart-umbrella-station

#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>

// RFID 핀 설정
#define RST_PIN 9
#define SS_PIN 10

// LED 핀 설정
#define RED_LED 6
#define GREEN_LED 7
#define BLUE_LED 8

// 서보모터 & 부저 핀
#define SERVO_PIN 3
#define BUZZER_PIN 2

// 레인센서 핀 (2핀 센서)
#define RAIN_SENSOR A0

MFRC522 mfrc522(SS_PIN, RST_PIN);
Servo doorServo;
bool isUmbrellaOut = false;
bool isRaining = false;
unsigned long lastRainCheck = 0;
int rainThreshold = 200;  // 비 감지 임계값 (조정 필요)

void setup() {
  Serial.begin(9600);
  
  // RFID 초기화
  SPI.begin();
  mfrc522.PCD_Init();
  
  // 핀 설정
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(BLUE_LED, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  // 서보모터 초기화
  doorServo.attach(SERVO_PIN);
  doorServo.write(0);
  
  // 시작 상태
  allLEDsOff();
  digitalWrite(BLUE_LED, HIGH);
  playStartupSound();
  
  Serial.println("╔══════════════════════════╗");
  Serial.println("║    🏢 스마트 우산 보관함    ║");
  Serial.println("║   🌧️ 2핀 레인센서       ║");
  Serial.println("╚══════════════════════════╝");
  
  // 초기 센서 값 측정
  int initialValue = analogRead(RAIN_SENSOR);
  Serial.print("초기 센서 값: ");
  Serial.println(initialValue);
  Serial.println("(이 값을 기준으로 임계값을 조정하세요)");
  Serial.println("📱 카드를 태그하세요");
}

void loop() {
  // 레인센서 확인 (1초마다)
  if (millis() - lastRainCheck > 1000) {
    checkRain();
    lastRainCheck = millis();
  }
  
  // RFID 카드 확인
  if (mfrc522.PICC_IsNewCardPresent()) {
    if (mfrc522.PICC_ReadCardSerial()) {
      
      playCardDetected();
      
      if (!isUmbrellaOut) {
        // === 우산 대여 ===
        allLEDsOff();
        digitalWrite(RED_LED, HIGH);
        
        Serial.println("🎉 카드 인식!");
        
        if (isRaining) {
          Serial.println("🌧️ 비가 오고 있어요! 우산 가져가세요!");
          playRainAlert();
        }
        
        Serial.println("🚪 문을 여는 중...");
        doorServo.write(90);
        delay(1000);
        
        playRentalSound();
        Serial.println("☂️  우산을 가져가세요!");
        delay(4000);
        
        Serial.println("🚪 문을 닫는 중...");
        doorServo.write(0);
        isUmbrellaOut = true;
        
      } else {
        // === 우산 반납 ===
        allLEDsOff();
        digitalWrite(GREEN_LED, HIGH);
        
        Serial.println("🎉 반납 카드 인식!");
        doorServo.write(90);
        delay(5000);
        doorServo.write(0);
        
        playReturnSound();
        Serial.println("✅ 반납 완료!");
        
        delay(2000);
        allLEDsOff();
        digitalWrite(BLUE_LED, HIGH);
        isUmbrellaOut = false;
      }
      
      Serial.println("━━━━━━━━━━━━━━━━━━━━━━━━━━");
      delay(1500);
      mfrc522.PICC_HaltA();
      mfrc522.PCD_StopCrypto1();
    }
  }
}

void checkRain() {
  int rainValue = analogRead(RAIN_SENSOR);
  bool currentRainState = (rainValue > rainThreshold);
  
  // 센서값 모니터링
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint > 3000) {
    Serial.print("🌦️ 센서 값: ");
    Serial.print(rainValue);
    if (currentRainState) {
      Serial.println(" (비 감지!)");
    } else {
      Serial.println(" (건조함)");
    }
    lastPrint = millis();
  }
  
  // 비 상태 변화 감지
  if (currentRainState && !isRaining) {
    isRaining = true;
    Serial.println("🌧️ 비가 시작되었어요!");
    
    if (!isUmbrellaOut) {
      Serial.println("☂️ 우산이 필요해요!");
      playRainAlert();
      rainLEDEffect();
    }
    
  } else if (!currentRainState && isRaining) {
    isRaining = false;
    Serial.println("🌤️ 비가 그쳤어요!");
    
    if (!isUmbrellaOut) {
      allLEDsOff();
      digitalWrite(BLUE_LED, HIGH);
    }
  }
}

void rainLEDEffect() {
  for (int i = 0; i < 3; i++) {
    allLEDsOff();
    delay(200);
    digitalWrite(BLUE_LED, HIGH);
    delay(200);
  }
}

void playRainAlert() {
  tone(BUZZER_PIN, 400, 200);
  delay(250);
  tone(BUZZER_PIN, 600, 200);
  delay(300);
}

void allLEDsOff() {
  digitalWrite(RED_LED, LOW);
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(BLUE_LED, LOW);
}

// 기존 부저 함수들은 동일...
void playStartupSound() {
  tone(BUZZER_PIN, 262, 250);
  delay(300);
  tone(BUZZER_PIN, 330, 250);
  delay(300);
  tone(BUZZER_PIN, 392, 250);
  delay(300);
  tone(BUZZER_PIN, 523, 400);
  delay(500);
}

void playCardDetected() {
  tone(BUZZER_PIN, 800, 150);
  delay(200);
}

void playRentalSound() {
  tone(BUZZER_PIN, 659, 200);
  delay(250);
  tone(BUZZER_PIN, 784, 200);
  delay(250);
  tone(BUZZER_PIN, 880, 300);
  delay(400);
}

void playReturnSound() {
  tone(BUZZER_PIN, 523, 150);
  delay(200);
  tone(BUZZER_PIN, 659, 150);
  delay(200);
  tone(BUZZER_PIN, 784, 150);
  delay(200);
  tone(BUZZER_PIN, 1047, 300);
  delay(400);
}
