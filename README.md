/*
   ==========================================
   ESP32 - Detector de Chuva + ThingSpeak
   ==========================================

   TASK 0 -> Automação
   TASK 1 -> WiFi + Cloud

   FUNÇÕES:
   - 3 sensores touch
   - Média móvel 15 amostras
   - Servo automático
   - ThingSpeak
   - Reconexão automática WiFi
   - Watchdog separado
   - LED interno indica WiFi ONLINE

   LED INTERNO:
   ONLINE  -> ACESO
   OFFLINE -> APAGADO
*/

#include <WiFi.h>
#include <HTTPClient.h>
#include <ESP32Servo.h>
#include <esp_task_wdt.h>

// ==========================================
// PINOS
// ==========================================

#define LED_CHUVA      18
#define LED_WIFI       2
#define SERVO_PIN      19

// ==========================================
// WIFI
// ==========================================

const char* ssid = "LUIZA";
const char* password = "32341098ltu";

// WRITE API KEY
String apiKey = "ODK6XRO1ZWS9CN52";

// ==========================================
// SERVO
// ==========================================

Servo servoMotor;

// ==========================================
// TOUCH
// ==========================================

// T8 = GPIO33
// T9 = GPIO32
// T7 = GPIO27

int sensores[3] = {T8, T9, T7};

// ==========================================
// CONFIG
// ==========================================

const int limite = 600;

const int amostras = 15;

const unsigned long intervaloLeitura = 50;

const unsigned long intervaloThingSpeak = 15000;

// ==========================================
// MEDIA MOVEL
// ==========================================

int buffer[3][amostras];

long soma[3] = {0, 0, 0};

int media[3];

bool ativo[3];

int indice = 0;

// ==========================================
// ESTADO GLOBAL
// ==========================================

volatile bool chuvaDetectada = false;

volatile int estadoLED = 0;

// ==========================================
// TIMERS
// ==========================================

unsigned long tempoLeitura = 0;

unsigned long tempoThingSpeak = 0;

// ==========================================
// TASK HANDLES
// ==========================================

TaskHandle_t TaskAutomacao;

TaskHandle_t TaskWiFi;

// ==========================================
// THINGSPEAK
// ==========================================

void enviarThingSpeak(int valor) {

  if (WiFi.status() == WL_CONNECTED) {

    HTTPClient http;

    String url =
      "http://api.thingspeak.com/update?api_key=" +
      apiKey +
      "&field1=" +
      String(valor);

    http.begin(url);

    int httpCode = http.GET();

    Serial.print("ThingSpeak HTTP: ");
    Serial.println(httpCode);

    http.end();
  }
}

// ==========================================
// TASK WIFI
// ==========================================

void taskWiFi(void *pvParameters) {

  // WATCHDOG DA TASK WIFI
  esp_task_wdt_add(NULL);

  while (true) {

    esp_task_wdt_reset();

    // ======================================
    // VERIFICA WIFI
    // ======================================

    if (WiFi.status() != WL_CONNECTED) {

      digitalWrite(LED_WIFI, LOW);

      Serial.println("WiFi OFFLINE");

      WiFi.disconnect();

      WiFi.begin(ssid, password);

      unsigned long inicio = millis();

      // tenta conectar
      while (WiFi.status() != WL_CONNECTED &&
             millis() - inicio < 10000) {

        esp_task_wdt_reset();

        Serial.print(".");

        delay(500);
      }

      if (WiFi.status() == WL_CONNECTED) {

        Serial.println("");
        Serial.println("WiFi conectado");

        digitalWrite(LED_WIFI, HIGH);

        Serial.println(WiFi.localIP());

      } else {

        Serial.println("");
        Serial.println("Falha WiFi");

      }
    }

    // ======================================
    // ENVIO CLOUD
    // ======================================

    if (millis() - tempoThingSpeak >= intervaloThingSpeak) {

      tempoThingSpeak = millis();

      enviarThingSpeak(estadoLED);

      Serial.print("Status enviado: ");
      Serial.println(estadoLED);
    }

    vTaskDelay(1000 / portTICK_PERIOD_MS);
  }
}

// ==========================================
// TASK AUTOMACAO
// ==========================================

void taskAutomacao(void *pvParameters) {

  // WATCHDOG DA TASK AUTOMACAO
  esp_task_wdt_add(NULL);

  while (true) {

    esp_task_wdt_reset();

    unsigned long agora = millis();

    if (agora - tempoLeitura >= intervaloLeitura) {

      tempoLeitura = agora;

      int sensoresAtivos = 0;

      Serial.println("---------------");

      // ====================================
      // LEITURA SENSORES
      // ====================================

      for (int s = 0; s < 3; s++) {

        soma[s] -= buffer[s][indice];

        buffer[s][indice] = touchRead(sensores[s]);

        soma[s] += buffer[s][indice];

        media[s] = soma[s] / amostras;

        if (media[s] < limite) {

          ativo[s] = true;

          sensoresAtivos++;

        } else {

          ativo[s] = false;

        }

        // DEBUG

        Serial.print("Sensor ");
        Serial.print(s);

        Serial.print(" | Media: ");
        Serial.print(media[s]);

        Serial.print(" | ");

        if (ativo[s]) {

          Serial.println("CHUVA");

        } else {

          Serial.println("SECO");

        }
      }

      // ====================================
      // INDICE CIRCULAR
      // ====================================

      indice++;

      if (indice >= amostras) {

        indice = 0;
      }

      // ====================================
      // PROBABILIDADE
      // ====================================

      float probabilidade = sensoresAtivos * 33.33;

      Serial.print("Probabilidade: ");
      Serial.print(probabilidade);
      Serial.println("%");

      // ====================================
      // DECISAO
      // ====================================

      if (sensoresAtivos >= 2) {

        digitalWrite(LED_CHUVA, HIGH);

        estadoLED = 1;

        Serial.println(">>> CHUVA DETECTADA");

        if (!chuvaDetectada) {

          servoMotor.write(0);

          chuvaDetectada = true;

          Serial.println("Servo -> 0 graus");
        }

      } else {

        digitalWrite(LED_CHUVA, LOW);

        estadoLED = 0;

        Serial.println(">>> SEM CHUVA");

        if (chuvaDetectada) {

          servoMotor.write(90);

          chuvaDetectada = false;

          Serial.println("Servo -> 90 graus");
        }
      }
    }

    vTaskDelay(20 / portTICK_PERIOD_MS);
  }
}

// ==========================================
// SETUP
// ==========================================

void setup() {

  Serial.begin(115200);

  // ========================================
  // LEDS
  // ========================================

  pinMode(LED_CHUVA, OUTPUT);

  pinMode(LED_WIFI, OUTPUT);

  digitalWrite(LED_CHUVA, LOW);

  digitalWrite(LED_WIFI, LOW);

  // ========================================
  // SERVO
  // ========================================

  servoMotor.setPeriodHertz(50);

  servoMotor.attach(SERVO_PIN, 500, 2400);

  servoMotor.write(90);

  // ========================================
  // WIFI
  // ========================================

  WiFi.mode(WIFI_STA);

  // ========================================
  // WATCHDOG GLOBAL
  // ========================================

  esp_task_wdt_config_t wdt_config = {
    .timeout_ms = 15000,
    .idle_core_mask = (1 << portNUM_PROCESSORS) - 1,
    .trigger_panic = true
  };

  esp_task_wdt_init(&wdt_config);

  // ========================================
  // INICIALIZA BUFFERS
  // ========================================

  for (int s = 0; s < 3; s++) {

    for (int i = 0; i < amostras; i++) {

      buffer[s][i] = 0;
    }
  }

  // ========================================
  // TASK AUTOMACAO -> CORE 0
  // ========================================

  xTaskCreatePinnedToCore(
    taskAutomacao,
    "TaskAutomacao",
    10000,
    NULL,
    1,
    &TaskAutomacao,
    0);

  // ========================================
  // TASK WIFI -> CORE 1
  // ========================================

  xTaskCreatePinnedToCore(
    taskWiFi,
    "TaskWiFi",
    10000,
    NULL,
    1,
    &TaskWiFi,
    1);

  Serial.println("Sistema iniciado");
}

// ==========================================
// LOOP VAZIO
// ==========================================

void loop() {

  vTaskDelay(portMAX_DELAY);
}
