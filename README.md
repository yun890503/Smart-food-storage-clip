# Smart-food-storage-clip
#include <TM1637Display.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>

// === WiFi & LINE 設定 ===
const char* WIFI_SSID     = "WILLIAMS IPHONE";      // WiFi 名稱
const char* WIFI_PASSWORD = "20000503";             // WiFi 密碼

// Channel access token（建議之後在 LINE Developers 換掉這一組）
const char* CHANNEL_TOKEN = "2lozxJOvVLXD7lYR8T/SfT0SIfShfXuOrw7Nd0rHg3t9HZoTKJwmOaSH7Yvcgus/ZLzdpg2005w4A1SEMT9FFonU5ZnTR1N+75dard1O4oYoaukDEySHGlJbadLIs5LSIc2YOOsnl3TrDgZbpImYYgdB04t89/1O/w1cDnyilFU=";

// 你的 LINE User ID
const char* USER_ID       = "U5e7511e60c22086da3ae3b68b389766b";

// === 七段顯示器 ===
#define CLK 4
#define DIO 3
TM1637Display display(CLK, DIO);

// === 三色燈 ===
#define RED_LED    10
#define YELLOW_LED  9
#define GREEN_LED   2

// === 四個按鍵 ===
#define K1 8   // 天數＋
#define K2 5   // 天數－
#define K3 6   // 開始/暫停
#define K4 7   // 重置

// === 參數 ===
int  daysSet    = 0;       // 使用者設定的天數
int  daysRemain = 0;       // 剩餘的天數
bool isRunning  = false;   // 是否正在倒數

// 測試模式：1 秒當 1 天
unsigned long lastUpdate        = 0;
const unsigned long DAY_MILLIS  = 1000;   // 正式版可改成 86400000 (24 小時)

// WiFi 定期檢查用（避免每次 loop 都一直重連）
unsigned long lastWiFiCheck = 0;
const unsigned long WIFI_CHECK_INTERVAL = 30000;   // 每 30 秒檢查一次

// =================================================
// WiFi & LINE 函式
// =================================================

// 初次啟動用
void initWiFi() {
  Serial.println("連線到 WiFi...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 15000) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("WiFi 已連線");
    Serial.print("IP: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("初次連線 WiFi 失敗，之後會再嘗試重連。");
  }
}

// 發送前確認 WiFi 有連上，沒有就嘗試重連一次
bool ensureWiFiConnected() {
  if (WiFi.status() == WL_CONNECTED) {
    return true;
  }

  Serial.println("WiFi 未連線，嘗試重新連線...");
  WiFi.disconnect(true);
  delay(200);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 15000) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("WiFi 重新連線成功，IP: ");
    Serial.println(WiFi.localIP());
    return true;
  } else {
    Serial.println("WiFi 重新連線失敗！");
    return false;
  }
}

// 在 loop 裡可以偶爾叫一下，當背景健康檢查用（不會一直卡住）
void backgroundWiFiCheck() {
  if (millis() - lastWiFiCheck < WIFI_CHECK_INTERVAL) return;
  lastWiFiCheck = millis();

  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("[背景檢查] 偵測到 WiFi 掉線，嘗試重連...");
    ensureWiFiConnected();
  }
}

/**
 * 用 Flex Bubble 的形式發 LINE Push
 * title   : Bubble 上方標題，例如「食品即將到期」
 * message : 內容文字，例如「食物還有 7 天到期，請儘快食用。」
 * color   : 標題字的顏色，例如 "#4CAF50", "#FF9800", "#F44336"
 */
void sendLineBubble(const String& title, const String& message, const String& color) {
  // 1. 先確認 WiFi 有連上
  if (!ensureWiFiConnected()) {
    Serial.println("因為 WiFi 沒連上，取消發送 LINE 訊息。");
    return;
  }

  WiFiClientSecure client;
  client.setInsecure();   // 不驗證憑證，對 ESP32 最簡單

  Serial.println("準備連線到 LINE API...");
  if (!client.connect("api.line.me", 443)) {
    Serial.println("連線 LINE API 失敗 (client.connect 回傳 false)");
    client.stop();
    return;
  }
  Serial.println("已連上 LINE API。");

  String altText = "智慧保鮮夾提醒";  // 手機通知預覽文字

  // 建立 Flex Bubble JSON
  String body = "{";
  body += "\"to\":\"" + String(USER_ID) + "\",";
  body += "\"messages\":[{";
  body +=   "\"type\":\"flex\",";
  body +=   "\"altText\":\"" + altText + "\",";
  body +=   "\"contents\":{";
  body +=     "\"type\":\"bubble\",";
  body +=     "\"body\":{";
  body +=       "\"type\":\"box\",";
  body +=       "\"layout\":\"vertical\",";
  body +=       "\"spacing\":\"md\",";
  body +=       "\"contents\":[";
  // 標題
  body +=         "{";
  body +=           "\"type\":\"text\",";
  body +=           "\"text\":\"" + title + "\",";
  body +=           "\"weight\":\"bold\",";
  body +=           "\"size\":\"lg\",";
  body +=           "\"color\":\"" + color + "\"";
  body +=         "},";
  // 分隔線
  body +=         "{";
  body +=           "\"type\":\"separator\",";
  body +=           "\"margin\":\"md\"";
  body +=         "},";
  // 內容文字
  body +=         "{";
  body +=           "\"type\":\"text\",";
  body +=           "\"text\":\"" + message + "\",";
  body +=           "\"wrap\":true,";
  body +=           "\"margin\":\"md\",";
  body +=           "\"size\":\"sm\"";
  body +=         "}";
  body +=       "]";
  body +=     "}";
  body +=   "}";
  body += "}]";
  body += "}";

  // HTTP request
  client.println("POST /v2/bot/message/push HTTP/1.1");
  client.println("Host: api.line.me");
  client.println("Authorization: Bearer " + String(CHANNEL_TOKEN));
  client.println("Content-Type: application/json");
  client.print("Content-Length: ");
  client.println(body.length());
  client.println("Connection: close");
  client.println();
  client.print(body);

  Serial.println("已送出 LINE Flex Bubble：");
  Serial.println(title + " / " + message);
  Serial.println("等待 LINE 回應...");

  // 讀取 LINE API 的回應（方便 debug）
  unsigned long respStart = millis();
  while (millis() - respStart < 5000) {   // 最多等 5 秒
    while (client.available()) {
      String line = client.readStringUntil('\n');
      line.trim();
      if (line.length() > 0) {
        Serial.println(line);
      }
      respStart = millis(); // 只要有資料就延長等待時間
    }

    if (!client.connected()) {
      break;
    }
  }

  client.stop();   // ★ 關閉 TLS 連線，避免一直佔著資源
  Serial.println("已關閉與 LINE API 的連線。");
}

// =================================================
// Arduino 標準流程
// =================================================
void setup() {
  Serial.begin(115200);
  delay(500);

  display.setBrightness(0x0f, true);

  pinMode(RED_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  pinMode(K1, INPUT_PULLUP);
  pinMode(K2, INPUT_PULLUP);
  pinMode(K3, INPUT_PULLUP);
  pinMode(K4, INPUT_PULLUP);

  initWiFi();   // 先嘗試連 WiFi
  showSetting();
}

void loop() {
  readButtons();

  if (isRunning) {
    runCountdown();
  } else {
    showSetting();
  }

  // 偶爾做一下 WiFi 背景檢查，避免 AP 休眠後永遠斷線
  backgroundWiFiCheck();
}

// =================================================
// 按鍵讀取
// =================================================
void readButtons() {
  if (digitalRead(K1) == LOW && !isRunning) {
    daysSet++;
    Serial.print("天數＋1，目前設定：");
    Serial.println(daysSet);
    delay(200);
  }

  if (digitalRead(K2) == LOW && !isRunning) {
    if (daysSet > 0) daysSet--;
    Serial.print("天數－1，目前設定：");
    Serial.println(daysSet);
    delay(200);
  }

  if (digitalRead(K3) == LOW) {
    if (!isRunning) {
      daysRemain = daysSet;
      isRunning  = true;
      lastUpdate = millis();   // 重新開始計時
      Serial.println("開始倒數");
    } else {
      isRunning = false;
      Serial.println("暫停倒數");
    }
    delay(300);
  }

  if (digitalRead(K4) == LOW) {
    isRunning  = false;
    daysSet    = 0;
    daysRemain = 0;
    Serial.println("重置");
    delay(300);
  }
}

// =================================================
// 顯示設定中的天數
// =================================================
void showSetting() {
  updateLED(daysSet);
  display.showNumberDec(daysSet, true);
}

// =================================================
// 天數倒數（含 LINE Flex 推播）
// =================================================
void runCountdown() {
  if (millis() - lastUpdate >= DAY_MILLIS) {
    lastUpdate = millis();

    if (daysRemain > 0) {
      daysRemain--;

      Serial.print("剩餘：");
      Serial.println(daysRemain);
      display.showNumberDec(daysRemain, true);
      updateLED(daysRemain);

      // ★★★ 這裡做推播判斷（Bubble） ★★★
      if (daysRemain == 7) {
        sendLineBubble(
          "食品保存提醒",
          "食物還有 7 天到期，請儘快食用或安排料理 👍",
          "#4CAF50"   // 綠色
        );
      } else if (daysRemain == 1) {
        sendLineBubble(
          "⚠ 即將到期",
          "食物還有 1 天到期，建議今天或明天儘快食用。",
          "#FF9800"   // 橘色
        );
      }

    } else {
      // 過期
      daysRemain = 0;
      display.showNumberDec(0, true);
      updateLED(0);

      sendLineBubble(
        "❌ 食品已過期",
        "食物已超過保存期限，請勿繼續食用，並留意下次保存時間。",
        "#F44336"   // 紅色
      );

      // 避免一直發，停止倒數
      isRunning = false;
    }
  }
}

// =================================================
// LED 狀態：綠 >7 天，黃 1～7，紅過期/0
// =================================================
void updateLED(int d) {
  if (d > 7) {
    digitalWrite(GREEN_LED, HIGH);
    digitalWrite(YELLOW_LED, LOW);
    digitalWrite(RED_LED, LOW);
  }
  else if (d >= 1 && d <= 7) {
    digitalWrite(GREEN_LED, LOW);
    digitalWrite(YELLOW_LED, HIGH);
    digitalWrite(RED_LED, LOW);
  }
  else { // d == 0 或更小
    digitalWrite(GREEN_LED, LOW);
    digitalWrite(YELLOW_LED, LOW);
    digitalWrite(RED_LED, HIGH);
  }
}
