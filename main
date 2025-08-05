#include <WiFi.h>
#include <WebServer.h>
#include <DHT.h>
#include <time.h>
#include <Preferences.h>
#include <HTTPUpdateServer.h>
#include <ArduinoOTA.h>
#include <ArduinoJson.h>
#include <algorithm>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>

// Константы
#define DHTPIN 5
#define DHTTYPE DHT11
#define RAIN_SENSOR_PIN A0
#define HISTORY_SAVE_INTERVAL 5 * 60 * 1000 // 5 минут
#define WEB_UPDATE_INTERVAL 5000
#define HISTORY_SIZE 50
#define MAX_SSID_LENGTH 32
#define MAX_PASSWORD_LENGTH 64
#define CSRF_TOKEN_LENGTH 32
#define TELEGRAM_CHECK_INTERVAL 1000

// Telegram настройки (ЗАМЕНИТЕ НА СВОИ!)
#define TELEGRAM_BOT_TOKEN ""
#define TELEGRAM_CHAT_ID ""

// Структуры данных
struct WiFiSettings {
  char ssid[MAX_SSID_LENGTH];
  char password[MAX_PASSWORD_LENGTH];
};

struct OTASettings {
  char username[MAX_SSID_LENGTH];
  char password[MAX_PASSWORD_LENGTH];
};

struct SensorData {
  float temperature;
  float humidity;
  bool isRaining;
  int rainValue;
  int rainThreshold;
  String lastUpdate;
};

struct HistoryRecord {
  float temperature;
  float humidity;
  bool isRaining;
  String timestamp;
};

struct {
  HistoryRecord records[HISTORY_SIZE];
  int count = 0;
  int index = 0;
} sensorHistory;

// Глобальные объекты
WiFiSettings wifiSettings;
OTASettings otaSettings;
SensorData sensorData;
WebServer server(80);
DHT dht(DHTPIN, DHTTYPE);
Preferences preferences;
HTTPUpdateServer httpUpdater;
WiFiClientSecure secured_client;
UniversalTelegramBot bot(TELEGRAM_BOT_TOKEN, secured_client);

// Переменные состояния
unsigned long lastHistorySave = 0;
unsigned long lastWebUpdate = 0;
unsigned long lastTelegramCheck = 0;
int timeZoneOffset = 3;
bool isAPMode = false;
bool isWiFiConfigured = false;
bool shouldReboot = false;
String csrfToken;

// Прототипы функций
void initPreferences();
void saveWiFiSettings();
void saveOTASettings();
void connectWiFi();
void activateAPMode();
void configLocalTime();
void checkWiFi();
void readSensors();
void calibrateRainSensor();
void saveHistory();
String generateHTML();
void handleRoot();
void handleSensorData();
void handleHistoryData();
void handleSetTZ();
void handleCalibrate();
void handleSaveWiFi();
void handleSaveOTA();
void handleReset();
void setupOTA();
void setupWebServer();
void generateCsrfToken();
bool validateCsrf();
void handleTelegram();
void sendTelegramNotification(const String &message, const String &parse_mode = "");
String generateTelegramMenu();
String generateTelegramKeyboard();

void setup() {
  Serial.begin(115200);
  
  // Инициализация настроек
  initPreferences();
  
  // Генерация CSRF-токена
  generateCsrfToken();
  
  // Настройка пинов
  pinMode(RAIN_SENSOR_PIN, INPUT);
  dht.begin();
  
  // Подключение WiFi
  connectWiFi();
  
  // Настройка времени
  if (WiFi.status() == WL_CONNECTED) {
    configLocalTime();
    secured_client.setInsecure(); // Для простоты отключаем проверку сертификата
  }
  
  // Калибровка датчика дождя
  calibrateRainSensor();
  
  // Настройка OTA и веб-сервера
  setupOTA();
  setupWebServer();
  
  Serial.println("Система инициализирована!");
  sendTelegramNotification("🚀 *Метеостанция запущена!*\nIP: " + WiFi.localIP().toString() + "\nИспользуйте кнопку *Меню* для управления", "Markdown");
}

void loop() {
  ArduinoOTA.handle();
  checkWiFi();
  server.handleClient();
  
  // Обработка Telegram сообщений
  if (millis() - lastTelegramCheck > TELEGRAM_CHECK_INTERVAL && WiFi.status() == WL_CONNECTED) {
    handleTelegram();
    lastTelegramCheck = millis();
  }
  
  if (shouldReboot) {
    sendTelegramNotification("🔁 *Метеостанция перезагружается...*", "Markdown");
    Serial.println("Перезагрузка системы...");
    delay(1000);
    ESP.restart();
  }
  
  // Автоматическое сохранение в историю
  if (millis() - lastHistorySave > HISTORY_SAVE_INTERVAL) {
    readSensors();
    saveHistory();
    lastHistorySave = millis();
    
    // Уведомление о дожде
    static bool lastRainStatus = false;
    if (sensorData.isRaining != lastRainStatus) {
      if (sensorData.isRaining) {
        sendTelegramNotification("🌧️ *Внимание! Начался дождь!*\nТемпература: " + String(sensorData.temperature, 1) + "°C\nВлажность: " + String(sensorData.humidity, 1) + "%", "Markdown");
      } else {
        sendTelegramNotification("☀️ *Дождь закончился*\nТемпература: " + String(sensorData.temperature, 1) + "°C\nВлажность: " + String(sensorData.humidity, 1) + "%", "Markdown");
      }
      lastRainStatus = sensorData.isRaining;
    }
  }
  
  delay(10);
}

// ========== Telegram Functions ==========
void handleTelegram() {
  int newMessages = bot.getUpdates(bot.last_message_received + 1);

  for (int i = 0; i < newMessages; i++) {
    String chat_id = String(bot.messages[i].chat_id);
    if (chat_id != TELEGRAM_CHAT_ID) {
      bot.sendMessage(chat_id, "⛔ Доступ запрещен", "");
      continue;
    }

    String text = bot.messages[i].text;
    Serial.println("Telegram: " + text);

    if (text == "/start" || text == "/help" || text == "Меню") {
      bot.sendMessageWithReplyKeyboard(chat_id, generateTelegramMenu(), "Markdown", generateTelegramKeyboard(), true);
    }
    else if (text == "📊 Текущие показания" || text == "/status") {
      readSensors();
      String message = "📊 *Текущие показания*\n\n";
      message += "🌡️ Температура: *" + String(sensorData.temperature, 1) + " °C*\n";
      message += "💧 Влажность: *" + String(sensorData.humidity, 1) + " %*\n";
      message += sensorData.isRaining ? "🌧️ Состояние: *Идет дождь*\n" : "☀️ Состояние: *Без осадков*\n";
      message += "📶 Сигнал WiFi: " + String(WiFi.RSSI()) + " dBm\n";
      message += "🕒 Последнее обновление: " + sensorData.lastUpdate;
      bot.sendMessage(chat_id, message, "Markdown");
    }
    else if (text == "⏳ История данных" || text == "/history") {
      String message = "⏳ *Последние 5 измерений*\n\n";
      int count = min(5, sensorHistory.count);
      
      for (int i = 0; i < count; i++) {
        int idx = (sensorHistory.index - count + i + HISTORY_SIZE) % HISTORY_SIZE;
        message += "🕒 " + sensorHistory.records[idx].timestamp + "\n";
        message += "🌡️ " + String(sensorHistory.records[idx].temperature, 1) + " °C  ";
        message += "💧 " + String(sensorHistory.records[idx].humidity, 1) + " %\n";
        message += sensorHistory.records[idx].isRaining ? "🌧️ *Дождь*\n\n" : "☀️ *Сухо*\n\n";
      }
      bot.sendMessage(chat_id, message, "Markdown");
    }
    else if (text == "🔧 Калибровка" || text == "/calibrate") {
      calibrateRainSensor();
      bot.sendMessage(chat_id, "🔧 *Датчик дождя откалиброван*\nНовый порог: " + String(sensorData.rainThreshold), "Markdown");
    }
    else if (text == "🔄 Перезагрузка" || text == "/reboot") {
      bot.sendMessage(chat_id, "🔁 *Перезагрузка системы...*", "Markdown");
      shouldReboot = true;
    }
    else {
      bot.sendMessage(chat_id, "❌ Неизвестная команда. Нажмите кнопку *Меню*", "Markdown");
    }
  }
}

String generateTelegramMenu() {
  String menu = "📡 *Метеостанция - Главное меню*\n\n";
  menu += "Выберите действие:\n\n";
  menu += "📊 *Текущие показания* - актуальные данные с датчиков\n";
  menu += "⏳ *История данных* - последние измерения\n";
  menu += "🔧 *Калибровка* - калибровка датчика дождя\n";
  menu += "🔄 *Перезагрузка* - перезапуск системы\n\n";
  menu += "Для обновления меню нажмите кнопку *Меню*";
  return menu;
}

String generateTelegramKeyboard() {
  String keyboardJson = "[[\"📊 Текущие показания\", \"⏳ История данных\"],";
  keyboardJson += "[\"🔧 Калибровка\", \"🔄 Перезагрузка\"],";
  keyboardJson += "[\"Меню\"]]";
  return keyboardJson;
}

void sendTelegramNotification(const String &message, const String &parse_mode) {
  if (WiFi.status() == WL_CONNECTED) {
    bot.sendMessage(TELEGRAM_CHAT_ID, message, parse_mode);
  }
}

// ========== Настройки ==========
void initPreferences() {
  preferences.begin("meteo-station", false);
  
  // Загрузка WiFi настроек
  size_t ssidLen = preferences.getBytes("wifi_ssid", wifiSettings.ssid, sizeof(wifiSettings.ssid));
  size_t passLen = preferences.getBytes("wifi_pass", wifiSettings.password, sizeof(wifiSettings.password));
  isWiFiConfigured = (ssidLen > 0);
  
  if (!isWiFiConfigured) {
    strlcpy(wifiSettings.ssid, "MeteoStation-AP", sizeof(wifiSettings.ssid));
    strlcpy(wifiSettings.password, "meteo12345", sizeof(wifiSettings.password));
    saveWiFiSettings();
  }
  
  // Загрузка OTA настроек
  size_t otaUserLen = preferences.getBytes("ota_user", otaSettings.username, sizeof(otaSettings.username));
  size_t otaPassLen = preferences.getBytes("ota_pass", otaSettings.password, sizeof(otaSettings.password));
  
  if (otaUserLen == 0 || otaPassLen == 0) {
    strlcpy(otaSettings.username, "admin", sizeof(otaSettings.username));
    strlcpy(otaSettings.password, "meteo123", sizeof(otaSettings.password));
    saveOTASettings();
  }
}

void saveWiFiSettings() {
  preferences.putBytes("wifi_ssid", wifiSettings.ssid, strnlen(wifiSettings.ssid, sizeof(wifiSettings.ssid)));
  preferences.putBytes("wifi_pass", wifiSettings.password, strnlen(wifiSettings.password, sizeof(wifiSettings.password)));
  isWiFiConfigured = true;
}

void saveOTASettings() {
  preferences.putBytes("ota_user", otaSettings.username, strnlen(otaSettings.username, sizeof(otaSettings.username)));
  preferences.putBytes("ota_pass", otaSettings.password, strnlen(otaSettings.password, sizeof(otaSettings.password)));
  ArduinoOTA.setPassword(otaSettings.password);
}

// ========== WiFi Functions ==========
void connectWiFi() {
  if (!isWiFiConfigured) {
    activateAPMode();
    return;
  }

  WiFi.mode(WIFI_STA);
  WiFi.begin(wifiSettings.ssid, wifiSettings.password);
  
  Serial.print("Подключение к ");
  Serial.println(wifiSettings.ssid);
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✅ WiFi подключен: " + WiFi.localIP().toString());
    isAPMode = false;
  } else {
    Serial.println("\n❌ Не удалось подключиться к WiFi");
    activateAPMode();
  }
}

void activateAPMode() {
  Serial.println("Активация режима точки доступа...");
  WiFi.mode(WIFI_AP);
  WiFi.softAP(wifiSettings.ssid, wifiSettings.password);
  isAPMode = true;
  
  Serial.print("Точка доступа: ");
  Serial.println(wifiSettings.ssid);
  Serial.print("Пароль: ");
  Serial.println(wifiSettings.password);
  Serial.println("IP адрес: " + WiFi.softAPIP().toString());
}

void checkWiFi() {
  static unsigned long lastCheck = 0;
  if (millis() - lastCheck > 10000) { // Проверка каждые 10 секунд
    if (WiFi.status() != WL_CONNECTED && !isAPMode) {
      Serial.println("📶 Переподключение к WiFi...");
      connectWiFi();
    }
    lastCheck = millis();
  }
}

// ========== Sensor Functions ==========
void readSensors() {
  static unsigned long lastRead = 0;
  if (millis() - lastRead < 2000) return; // Чтение не чаще чем раз в 2 секунды
  
  // Медианный фильтр для DHT (3 измерения)
  float tempBuffer[3];
  float humBuffer[3];
  
  for (int i = 0; i < 3; i++) {
    tempBuffer[i] = dht.readTemperature();
    humBuffer[i] = dht.readHumidity();
    delay(100);
  }
  
  // Сортировка для медианы
  std::sort(tempBuffer, tempBuffer + 3);
  std::sort(humBuffer, humBuffer + 3);
  
  sensorData.temperature = tempBuffer[1];
  sensorData.humidity = humBuffer[1];
  
  // Чтение датчика дождя
  sensorData.rainValue = analogRead(RAIN_SENSOR_PIN);
  sensorData.isRaining = sensorData.rainValue > sensorData.rainThreshold;
  
  // Получение времени
  struct tm timeinfo;
  if (getLocalTime(&timeinfo)) {
    char timeStr[20];
    strftime(timeStr, sizeof(timeStr), "%H:%M %d.%m", &timeinfo);
    sensorData.lastUpdate = String(timeStr);
  } else {
    sensorData.lastUpdate = "--:-- --.--";
  }
  
  lastRead = millis();
}

void calibrateRainSensor() {
  int sum = 0;
  for (int i = 0; i < 10; i++) {
    sum += analogRead(RAIN_SENSOR_PIN);
    delay(100);
  }
  sensorData.rainThreshold = (sum / 10) + 100;
  Serial.println("Датчик дождя откалиброван. Порог: " + String(sensorData.rainThreshold));
}

void saveHistory() {
  // Добавляем запись в историю
  if (sensorHistory.count < HISTORY_SIZE) {
    sensorHistory.count++;
  }
  
  // Запись в текущий индекс
  sensorHistory.records[sensorHistory.index] = {
    sensorData.temperature,
    sensorData.humidity,
    sensorData.isRaining,
    sensorData.lastUpdate
  };
  
  // Обновление индекса
  sensorHistory.index = (sensorHistory.index + 1) % HISTORY_SIZE;
  
  Serial.println("Данные сохранены в историю: " + sensorData.lastUpdate);
}

// ========== Security Functions ==========
void generateCsrfToken() {
  const char charset[] = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
  csrfToken = "";
  
  for (int i = 0; i < CSRF_TOKEN_LENGTH; i++) {
    csrfToken += charset[random(0, sizeof(charset) - 1)];
  }
}

bool validateCsrf() {
  if (server.method() == HTTP_GET) return true;
  return server.hasArg("csrf") && server.arg("csrf") == csrfToken;
}

// ========== Web Interface ==========
String generateHTML() {
  // Динамический расчет размера HTML
  size_t htmlSize = 7000 + sensorHistory.count * 120;
  String html;
  html.reserve(htmlSize);
  
  // HTML Head
  html = "<!DOCTYPE html><html lang=\"ru\"><head>";
  html += "<meta charset=\"UTF-8\">";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">";
  html += "<title>Метеостанция</title>";
  html += "<link href=\"https://fonts.googleapis.com/css2?family=Montserrat:wght@400;600;700&display=swap\" rel=\"stylesheet\">";
  html += "<link rel=\"stylesheet\" href=\"https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css\">";
  html += "<script src=\"https://cdn.jsdelivr.net/npm/chart.js\"></script>";
  html += "<style>";
  html += ":root {";
  html += "--primary: #4361ee;";
  html += "--secondary: #3f37c9;";
  html += "--accent: #4895ef;";
  html += "--danger: #f72585;";
  html += "--success: #4cc9f0;";
  html += "--warning: #f8961e;";
  html += "--light: #f8f9fa;";
  html += "--dark: #212529;";
  html += "--gray: #6c757d;";
  html += "}";
  
  html += "* { box-sizing: border-box; margin: 0; padding: 0; }";
  html += "body { font-family: 'Montserrat', sans-serif; background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%); color: var(--dark); min-height: 100vh; }";
  html += ".container { max-width: 1200px; margin: 0 auto; padding: 20px; }";
  html += "header { text-align: center; padding: 30px 0; margin-bottom: 30px; }";
  html += "header h1 { font-size: 2.5rem; margin-bottom: 10px; color: var(--primary); font-weight: 700; }";
  html += "header p { font-size: 1.1rem; color: var(--gray); }";
  html += ".dashboard { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 20px; margin-bottom: 30px; }";
  html += ".card { background: white; border-radius: 15px; padding: 25px; box-shadow: 0 10px 20px rgba(0,0,0,0.1); transition: transform 0.3s, box-shadow 0.3s; }";
  html += ".card:hover { transform: translateY(-5px); box-shadow: 0 15px 30px rgba(0,0,0,0.15); }";
  html += ".card-header { display: flex; align-items: center; margin-bottom: 20px; padding-bottom: 15px; border-bottom: 1px solid rgba(0,0,0,0.05); }";
  html += ".card-header i { font-size: 1.8rem; margin-right: 15px; color: var(--accent); }";
  html += ".card-header h2 { font-size: 1.3rem; font-weight: 600; color: var(--primary); }";
  html += ".card-body { display: flex; flex-direction: column; }";
  html += ".card-value { font-size: 2.5rem; font-weight: 700; margin: 10px 0; color: var(--secondary); }";
  html += ".card-status { display: inline-block; padding: 8px 15px; border-radius: 20px; font-weight: 600; color: white; margin-top: 10px; }";
  html += ".status-rain { background: linear-gradient(to right, var(--accent), var(--primary)); }";
  html += ".status-dry { background: linear-gradient(to right, var(--warning), var(--danger)); }";
  html += ".card-description { color: var(--gray); font-size: 0.9rem; margin-top: 5px; }";
  html += ".controls { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 20px; margin-bottom: 30px; }";
  html += ".control-panel { background: white; border-radius: 15px; padding: 25px; box-shadow: 0 10px 20px rgba(0,0,0,0.1); }";
  html += ".control-panel h3 { font-size: 1.3rem; margin-bottom: 20px; color: var(--primary); font-weight: 600; }";
  html += ".form-group { margin-bottom: 15px; }";
  html += ".form-group label { display: block; margin-bottom: 8px; font-weight: 600; color: var(--dark); }";
  html += ".form-control { width: 100%; padding: 12px 15px; border: 1px solid #ddd; border-radius: 8px; font-size: 1rem; transition: border 0.3s; }";
  html += ".form-control:focus { outline: none; border-color: var(--accent); }";
  html += ".btn { display: inline-block; padding: 12px 25px; background: var(--primary); color: white; border: none; border-radius: 8px; font-size: 1rem; font-weight: 600; cursor: pointer; transition: background 0.3s, transform 0.2s; text-align: center; }";
  html += ".btn:hover { background: var(--secondary); transform: translateY(-2px); }";
  html += ".btn-block { display: block; width: 100%; }";
  html += ".btn-danger { background: var(--danger); }";
  html += ".btn-danger:hover { background: #d1144a; }";
  html += ".info-bar { display: flex; justify-content: space-between; background: white; padding: 15px 25px; border-radius: 10px; margin-bottom: 20px; box-shadow: 0 5px 15px rgba(0,0,0,0.05); }";
  html += ".info-item { display: flex; align-items: center; }";
  html += ".info-item i { margin-right: 8px; color: var(--accent); }";
  html += ".alert { padding: 15px; border-radius: 10px; margin-bottom: 20px; background: #fff3cd; color: #856404; border-left: 5px solid #ffeeba; }";
  html += ".alert-warning { background: #fff3cd; color: #856404; border-left-color: #ffeeba; }";
  html += ".alert-danger { background: #f8d7da; color: #721c24; border-left-color: #f5c6cb; }";
  html += "table { width: 100%; border-collapse: collapse; margin-bottom: 15px; }";
  html += "th, td { padding: 10px; text-align: left; border-bottom: 1px solid #ddd; }";
  html += "tr:nth-child(even) { background-color: #f9f9f9; }";
  html += ".chart-container { height: 300px; margin-bottom: 20px; }";
  html += ".history-container { max-height: 300px; overflow-y: auto; margin-bottom: 15px; }";
  html += "footer { text-align: center; padding: 20px 0; color: var(--gray); font-size: 0.9rem; }";
  html += "@media (max-width: 768px) { .dashboard, .controls { grid-template-columns: 1fr; } .info-bar { flex-direction: column; gap: 10px; } }";
  html += "@media (pointer: coarse) { .btn { padding: 15px 30px; min-height: 50px; } }";
  html += "</style></head><body>";
  html += "<div class=\"container\">";
  
  // Header
  html += "<header><h1><i class=\"fas fa-cloud-sun\"></i> Умная метеостанция</h1>";
  html += "<p>Мониторинг погодных условий в реальном времени</p></header>";
  
  // Alert if in AP mode
  if (isAPMode) {
    html += "<div class=\"alert alert-warning\">";
    html += "<h3><i class=\"fas fa-exclamation-triangle\"></i> Режим настройки WiFi</h3>";
    html += "<p>Устройство не подключено к WiFi. Пожалуйста, настройте подключение.</p>";
    html += "</div>";
  }

  // Info bar
  html += "<div class=\"info-bar\">";
  html += "<div class=\"info-item\"><i class=\"fas fa-wifi\"></i> ";
  html += isAPMode ? WiFi.softAPIP().toString() : WiFi.localIP().toString();
  html += "</div>";
  html += "<div class=\"info-item\"><i class=\"fas fa-clock\"></i> Последнее обновление: ";
  html += sensorData.lastUpdate;
  html += "</div>";
  html += "<div class=\"info-item\"><i class=\"fas fa-signal\"></i> ";
  html += isAPMode ? "Точка доступа" : String(WiFi.RSSI()) + " dBm";
  html += "</div>";
  html += "</div>";

  // Dashboard cards
  html += "<div class=\"dashboard\">";
  html += "<div class=\"card temperature\"><div class=\"card-header\"><i class=\"fas fa-thermometer-half\"></i><h2>Температура</h2></div>";
  html += "<div class=\"card-body\"><div class=\"card-value\">";
  html += String(sensorData.temperature, 1);
  html += " °C</div><p class=\"card-description\">Текущая температура окружающей среды</p></div></div>";
  
  html += "<div class=\"card humidity\"><div class=\"card-header\"><i class=\"fas fa-tint\"></i><h2>Влажность</h2></div>";
  html += "<div class=\"card-body\"><div class=\"card-value\">";
  html += String(sensorData.humidity, 1);
  html += " %</div><p class=\"card-description\">Относительная влажность воздуха</p></div></div>";
  
  html += "<div class=\"card rain\"><div class=\"card-header\"><i class=\"fas fa-cloud-rain\"></i><h2>Дождь</h2></div>";
  html += "<div class=\"card-body\"><div class=\"card-value\">";
  html += String(sensorData.rainValue);
  html += "</div><div class=\"card-status ";
  html += sensorData.isRaining ? "status-rain" : "status-dry";
  html += "\">";
  html += sensorData.isRaining ? "<i class=\"fas fa-umbrella\"></i> Идёт дождь" : "<i class=\"fas fa-sun\"></i> Без осадков";
  html += "</div><p class=\"card-description\">Порог: ";
  html += String(sensorData.rainThreshold);
  html += "</p></div></div></div>";
  
  // Control panels
  html += "<div class=\"controls\">";
  
  // WiFi Settings panel
  html += "<div class=\"control-panel\"><h3><i class=\"fas fa-wifi\"></i> Настройки WiFi</h3>";
  html += "<form action=\"/savewifi\" method=\"post\">";
  html += "<input type=\"hidden\" name=\"csrf\" value=\"" + csrfToken + "\">";
  html += "<div class=\"form-group\"><label for=\"ssid\">Имя сети (SSID)</label>";
  html += "<input type=\"text\" class=\"form-control\" id=\"ssid\" name=\"ssid\" value=\"";
  html += wifiSettings.ssid;
  html += "\" maxlength=\"" + String(MAX_SSID_LENGTH - 1) + "\" required></div>";
  html += "<div class=\"form-group\"><label for=\"password\">Пароль</label>";
  html += "<input type=\"password\" class=\"form-control\" id=\"password\" name=\"password\" value=\"";
  html += wifiSettings.password;
  html += "\" maxlength=\"" + String(MAX_PASSWORD_LENGTH - 1) + "\" placeholder=\"Введите пароль\"></div>";
  html += "<button type=\"submit\" class=\"btn btn-block\"><i class=\"fas fa-save\"></i> Сохранить</button>";
  html += "</form></div>";
  
  // Time and Sensors panel
  html += "<div class=\"control-panel\"><h3><i class=\"fas fa-cog\"></i> Системные настройки</h3>";
  html += "<form action=\"/settz\" method=\"get\">";
  html += "<div class=\"form-group\"><label for=\"tz\">Часовой пояс</label>";
  html += "<select class=\"form-control\" name=\"tz\" id=\"tz\">";
  
  for (int i = -12; i <= 14; i++) {
    html += "<option value=\"";
    html += String(i);
    html += "\"";
    if (i == timeZoneOffset) html += " selected";
    html += ">UTC";
    if (i >= 0) html += "+";
    html += String(i);
    html += "</option>";
  }
  
  html += "</select></div>";
  html += "<div class=\"form-group\"><label for=\"rain_threshold\">Порог дождя</label>";
  html += "<input type=\"number\" class=\"form-control\" id=\"rain_threshold\" name=\"rain_threshold\" value=\"";
  html += String(sensorData.rainThreshold);
  html += "\"></div>";
  html += "<button type=\"submit\" class=\"btn btn-block\"><i class=\"fas fa-clock\"></i> Обновить</button>";
  html += "</form>";
  html += "<form action=\"/calibrate\" method=\"get\" style=\"margin-top: 10px;\">";
  html += "<button type=\"submit\" class=\"btn btn-block\"><i class=\"fas fa-bolt\"></i> Калибровать датчик</button>";
  html += "</form></div>";
  
  // OTA and System panel
  html += "<div class=\"control-panel\"><h3><i class=\"fas fa-power-off\"></i> Система</h3>";
  html += "<form action=\"/saveota\" method=\"post\">";
  html += "<input type=\"hidden\" name=\"csrf\" value=\"" + csrfToken + "\">";
  html += "<div class=\"form-group\"><label for=\"ota_user\">OTA Логин</label>";
  html += "<input type=\"text\" class=\"form-control\" id=\"ota_user\" name=\"ota_user\" value=\"";
  html += otaSettings.username;
  html += "\" maxlength=\"" + String(MAX_SSID_LENGTH - 1) + "\" required></div>";
  html += "<div class=\"form-group\"><label for=\"ota_pass\">OTA Пароль</label>";
  html += "<input type=\"password\" class=\"form-control\" id=\"ota_pass\" name=\"ota_pass\" value=\"";
  html += otaSettings.password;
  html += "\" maxlength=\"" + String(MAX_PASSWORD_LENGTH - 1) + "\" required></div>";
  html += "<button type=\"submit\" class=\"btn btn-block\"><i class=\"fas fa-save\"></i> Сохранить</button>";
  html += "</form>";
  html += "<form action=\"/update\" method=\"get\" style=\"margin-top: 10px;\">";
  html += "<button type=\"submit\" class=\"btn btn-block\"><i class=\"fas fa-cloud-upload-alt\"></i> OTA Обновление</button>";
  html += "</form>";
  html += "<form action=\"/reset\" method=\"get\" style=\"margin-top: 10px;\">";
  html += "<button type=\"submit\" class=\"btn btn-block btn-danger\"><i class=\"fas fa-sync-alt\"></i> Перезагрузить</button>";
  html += "</form></div>";
  
  html += "</div>"; // Close controls div
  
  // History panel with chart
  html += "<div class=\"control-panel\" style=\"grid-column: 1 / -1;\"><h3><i class=\"fas fa-history\"></i> История измерений</h3>";
  html += "<div class=\"chart-container\"><canvas id=\"historyChart\"></canvas></div>";
  
  // History table
  html += "<div class=\"history-container\">";
  html += "<table><thead><tr><th>Время</th><th>Темп.</th><th>Влажн.</th><th>Дождь</th></tr></thead><tbody>";

  for (int i = 0; i < sensorHistory.count; i++) {
    int idx = (sensorHistory.index - sensorHistory.count + i + HISTORY_SIZE) % HISTORY_SIZE;
    html += "<tr>";
    html += "<td>"; html += sensorHistory.records[idx].timestamp; html += "</td>";
    html += "<td>"; html += String(sensorHistory.records[idx].temperature, 1); html += " °C</td>";
    html += "<td>"; html += String(sensorHistory.records[idx].humidity, 1); html += " %</td>";
    html += "<td>"; 
    html += sensorHistory.records[idx].isRaining ? "Да" : "Нет"; 
    html += "</td>";
    html += "</tr>";
  }

  html += "</tbody></table></div>";
  html += "<button onclick=\"location.reload()\" class=\"btn btn-block\"><i class=\"fas fa-sync-alt\"></i> Обновить</button>";
  html += "</div>";
  
  // Footer
  html += "<footer><p><i class=\"fas fa-code\"></i> Умная метеостанция © 2023 | Версия 2.9</p></footer>";
  html += "</div>"; // Close container
  
  // JavaScript
  html += "<script>";
  html += "const historyChartConfig = {";
  html += "type: 'line',";
  html += "data: {";
  html += "datasets: [{";
  html += "label: 'Температура (°C)',";
  html += "borderColor: '#4361ee',";
  html += "backgroundColor: 'rgba(67, 97, 238, 0.1)',";
  html += "borderWidth: 2,";
  html += "yAxisID: 'y'";
  html += "}, {";
  html += "label: 'Влажность (%)',";
  html += "borderColor: '#4cc9f0',";
  html += "backgroundColor: 'rgba(76, 201, 240, 0.1)',";
  html += "borderWidth: 2,";
  html += "yAxisID: 'y1'";
  html += "}]";
  html += "},";
  html += "options: {";
  html += "responsive: true,";
  html += "maintainAspectRatio: false,";
  html += "interaction: { mode: 'index' },";
  html += "scales: {";
  html += "y: {";
  html += "type: 'linear',";
  html += "display: true,";
  html += "position: 'left',";
  html += "title: { display: true, text: 'Температура (°C)' },";
  html += "grid: { drawOnChartArea: true }";
  html += "},";
  html += "y1: {";
  html += "type: 'linear',";
  html += "display: true,";
  html += "position: 'right',";
  html += "min: 0,";
  html += "max: 100,";
  html += "title: { display: true, text: 'Влажность (%)' },";
  html += "grid: { drawOnChartArea: false }";
  html += "}";
  html += "}";
  html += "}";
  html += "};";
  
  html += "let historyChart = new Chart(";
  html += "document.getElementById('historyChart'),";
  html += "historyChartConfig";
  html += ");";
  
  html += "function updateSensorData() {";
  html += "fetch('/sensor-data').then(r => r.json()).then(data => {";
  html += "document.querySelector('.temperature .card-value').textContent = data.temp + ' °C';";
  html += "document.querySelector('.humidity .card-value').textContent = data.hum + ' %';";
  html += "const rainValue = document.querySelector('.rain .card-value');";
  html += "const rainStatus = document.querySelector('.rain .card-status');";
  html += "rainValue.textContent = data.rainValue;";
  html += "rainStatus.innerHTML = data.rain ? '<i class=\"fas fa-umbrella\"></i> Идёт дождь' : '<i class=\"fas fa-sun\"></i> Без осадков';";
  html += "rainStatus.className = data.rain ? 'card-status status-rain' : 'card-status status-dry';";
  html += "document.querySelector('.rain .card-description').textContent = 'Порог: ' + data.threshold;";
  html += "document.querySelector('.info-item:nth-child(2)').innerHTML = '<i class=\"fas fa-clock\"></i> Последнее обновление: ' + data.time;";
  html += "}).catch(e => console.error(e));";
  html += "}";
  
  html += "function updateHistory() {";
  html += "fetch('/history-data').then(r => r.json()).then(data => {";
  html += "const tbody = document.querySelector('tbody');";
  html += "tbody.innerHTML = '';";
  html += "data.history.forEach(record => {";
  html += "const row = document.createElement('tr');";
  html += "row.innerHTML = `";
  html += "<td>${record.time}</td>";
  html += "<td>${record.temp} °C</td>";
  html += "<td>${record.hum} %</td>";
  html += "<td>${record.rain ? 'Да' : 'Нет'}</td>";
  html += "`;";
  html += "tbody.appendChild(row);";
  html += "});";
  
  // Обновление графика с правильными данными
  html += "const labels = [];";
  html += "const tempData = [];";
  html += "const humData = [];";
  html += "for(let i = data.history.length - 1; i >= 0; i--) {";
  html += "labels.push(data.history[i].time);";
  html += "tempData.push(data.history[i].temp);";
  html += "humData.push(data.history[i].hum);";
  html += "}";
  html += "historyChart.data.labels = labels;";
  html += "historyChart.data.datasets[0].data = tempData;";
  html += "historyChart.data.datasets[1].data = humData;";
  html += "historyChart.update();";
  html += "}).catch(e => console.error(e));";
  html += "}";
  
  html += "document.addEventListener('DOMContentLoaded', () => {";
  html += "updateSensorData();";
  html += "updateHistory();";
  html += "setInterval(updateSensorData, 30000);";
  html += "setInterval(updateHistory, 60000);";
  html += "});";
  html += "</script></body></html>";

  return html;
}

// ========== Web Server Handlers ==========
void handleRoot() {
  if (millis() - lastWebUpdate > WEB_UPDATE_INTERVAL) {
    readSensors();
    server.sendHeader("Content-Type", "text/html; charset=UTF-8");
    server.send(200, "text/html", generateHTML());
    lastWebUpdate = millis();
  } else {
    server.send(429, "text/plain", "Пожалуйста, подождите...");
  }
}

void handleSensorData() {
  readSensors();
  
  DynamicJsonDocument doc(256);
  doc["temp"] = sensorData.temperature;
  doc["hum"] = sensorData.humidity;
  doc["rain"] = sensorData.isRaining;
  doc["rainValue"] = sensorData.rainValue;
  doc["threshold"] = sensorData.rainThreshold;
  doc["time"] = sensorData.lastUpdate;

  String json;
  serializeJson(doc, json);
  
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.send(200, "application/json", json);
}

void handleHistoryData() {
  DynamicJsonDocument doc(4096);
  JsonArray history = doc.createNestedArray("history");
  
  for (int i = 0; i < sensorHistory.count; i++) {
    int idx = (sensorHistory.index - sensorHistory.count + i + HISTORY_SIZE) % HISTORY_SIZE;
    JsonObject record = history.createNestedObject();
    record["time"] = sensorHistory.records[idx].timestamp;
    record["temp"] = sensorHistory.records[idx].temperature;
    record["hum"] = sensorHistory.records[idx].humidity;
    record["rain"] = sensorHistory.records[idx].isRaining;
  }
  
  String json;
  serializeJson(doc, json);
  
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.send(200, "application/json", json);
}

void handleSetTZ() {
  if (server.hasArg("tz")) {
    int tz = server.arg("tz").toInt();
    if (tz >= -12 && tz <= 14) {
      timeZoneOffset = tz;
      configLocalTime();
    }
  }
  if (server.hasArg("rain_threshold")) {
    sensorData.rainThreshold = server.arg("rain_threshold").toInt();
  }
  server.sendHeader("Location", "/");
  server.send(303);
}

void handleCalibrate() {
  calibrateRainSensor();
  server.sendHeader("Location", "/");
  server.send(303);
}

void handleSaveWiFi() {
  if (!validateCsrf()) {
    server.send(403, "text/plain", "Ошибка CSRF-токена");
    return;
  }
  
  if (server.method() == HTTP_POST) {
    String newSSID = server.arg("ssid");
    String newPass = server.arg("password");
    
    // Проверка длины
    if (newSSID.length() == 0 || newSSID.length() > MAX_SSID_LENGTH - 1) {
      server.send(400, "text/plain", "Ошибка: Некорректная длина SSID");
      return;
    }
    
    if (newPass.length() > MAX_PASSWORD_LENGTH - 1) {
      server.send(400, "text/plain", "Ошибка: Слишком длинный пароль");
      return;
    }
    
    strlcpy(wifiSettings.ssid, newSSID.c_str(), sizeof(wifiSettings.ssid));
    strlcpy(wifiSettings.password, newPass.c_str(), sizeof(wifiSettings.password));
    saveWiFiSettings();
    
    server.send(200, "text/plain", "Настройки WiFi сохранены! Перезагрузка...");
    shouldReboot = true;
    return;
  }
  server.send(400, "text/plain", "Ошибка: Неверный метод запроса");
}

void handleSaveOTA() {
  if (!validateCsrf()) {
    server.send(403, "text/plain", "Ошибка CSRF-токена");
    return;
  }
  
  if (server.method() == HTTP_POST) {
    String newUser = server.arg("ota_user");
    String newPass = server.arg("ota_pass");
    
    // Проверка длины
    if (newUser.length() == 0 || newUser.length() > MAX_SSID_LENGTH - 1) {
      server.send(400, "text/plain", "Ошибка: Некорректная длина логина");
      return;
    }
    
    if (newPass.length() == 0 || newPass.length() > MAX_PASSWORD_LENGTH - 1) {
      server.send(400, "text/plain", "Ошибка: Некорректная длина пароля");
      return;
    }
    
    strlcpy(otaSettings.username, newUser.c_str(), sizeof(otaSettings.username));
    strlcpy(otaSettings.password, newPass.c_str(), sizeof(otaSettings.password));
    saveOTASettings();
    
    server.sendHeader("Location", "/");
    server.send(303);
    return;
  }
  server.send(400, "text/plain", "Ошибка: Неверный метод запроса");
}

void handleReset() {
  server.send(200, "text/plain", "Перезагрузка системы...");
  shouldReboot = true;
}

// ========== Setup Functions ==========
void setupOTA() {
  ArduinoOTA.setHostname("MeteoStation");
  ArduinoOTA.setPassword(otaSettings.password);
  
  // Remove these lines as they're not supported:
  // ArduinoOTA.setVerify(true);
  // ArduinoOTA.setVerifyPassword("secure_verify_key");
  
  ArduinoOTA.onStart([]() {
    String type;
    if (ArduinoOTA.getCommand() == U_FLASH) {
      type = "sketch";
    } else { // U_SPIFFS
      type = "filesystem";
    }
    Serial.println("OTA Update Start: " + type);
  });
  
  ArduinoOTA.onEnd([]() {
    Serial.println("\nOTA Update End");
  });
  
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });
  
  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) Serial.println("Auth Failed");
    else if (error == OTA_BEGIN_ERROR) Serial.println("Begin Failed");
    else if (error == OTA_CONNECT_ERROR) Serial.println("Connect Failed");
    else if (error == OTA_RECEIVE_ERROR) Serial.println("Receive Failed");
    else if (error == OTA_END_ERROR) Serial.println("End Failed");
  });
  
  ArduinoOTA.begin();
}

void setupWebServer() {
  httpUpdater.setup(&server, "/update", otaSettings.username, otaSettings.password);
  
  server.on("/", handleRoot);
  server.on("/sensor-data", handleSensorData);
  server.on("/history-data", handleHistoryData);
  server.on("/settz", handleSetTZ);
  server.on("/calibrate", handleCalibrate);
  server.on("/savewifi", handleSaveWiFi);
  server.on("/saveota", handleSaveOTA);
  server.on("/reset", handleReset);
  
  server.begin();
}

void configLocalTime() {
  configTime(timeZoneOffset * 3600, 0, "pool.ntp.org", "time.nist.gov");
}
