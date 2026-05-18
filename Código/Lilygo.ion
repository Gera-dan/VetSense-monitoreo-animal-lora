// ============================================================
//  GATEWAY CELULAR — Sistema de Monitoreo Animal Inalámbrico
//  Identificador: VetSense Gateway
//  Tarjeta: LILYGO T-SIM7000G (ESP32 + módem SIM7000)
//  Descripción: Recibe el JSON del receptor vía ESP-NOW y lo
//               publica al broker HiveMQ por MQTT.
//               Tiene doble conectividad: intenta primero Wi-Fi
//               y cae a GPRS celular si no hay red disponible.
// ============================================================

// ── Librerías ───────────────────────────────────────────────
#define TINY_GSM_MODEM_SIM7000   // Indica a TinyGSM qué módem usar (SIM7000)
#include <TinyGsmClient.h>       // Cliente TCP/IP sobre el módem GSM
#include <PubSubClient.h>        // Cliente MQTT
#include <esp_now.h>             // Protocolo ESP-NOW (recepción desde el receptor)
#include <WiFi.h>                // Wi-Fi del ESP32
#include <WiFiClientSecure.h>    // Cliente TLS para MQTT sobre Wi-Fi (puerto 8883)

// ── Credenciales Wi-Fi (red de respaldo) ────────────────────
const char* ssid_wifi = "Galaxy";
const char* pass_wifi = "uoes9517";

// ── Configuración GPRS (Telcel México) ──────────────────────
const char apn[] = "internet.itelcel.com"; // APN de Telcel

// ── Configuración del broker MQTT HiveMQ Cloud ──────────────
// Se usa HiveMQ Cloud con TLS (puerto 8883) para mayor seguridad
const char* broker    = "17e730f43cbe4550abafa0b669aba544.s1.eu.hivemq.cloud";
const int   puerto    = 8883;          // Puerto MQTT con TLS
const char* mqtt_user = "GERAAAA";     // Usuario HiveMQ Cloud
const char* mqtt_pass = "123456Ab";    // Contraseña HiveMQ Cloud
const char* topic     = "vetsense/datos/animal"; // Tópico donde se publican los datos

// ── Pines del módem SIM7000 ──────────────────────────────────
#define PWR_PIN 4   // Pin de encendido del SIM7000 (pulso para encender/apagar)
#define PIN_TX  27  // TX del ESP32 → RX del SIM7000
#define PIN_RX  26  // RX del ESP32 ← TX del SIM7000

// ── Instancias de comunicación ───────────────────────────────
HardwareSerial   ModemSerial(1);         // UART1 para comunicarse con el SIM7000
TinyGsm          modem(ModemSerial);     // Control AT del módem via TinyGSM
WiFiClientSecure wifiClient;             // Cliente TLS para conexión Wi-Fi
TinyGsmClient    gsmClient(modem);       // Cliente TCP para conexión celular
PubSubClient     mqtt;                   // Cliente MQTT (se configura el transport después)

// ── Variables de estado ──────────────────────────────────────
String ultimoJSON    = "";    // Último JSON recibido por ESP-NOW (pendiente de publicar)
bool hayDatosNuevos  = false; // Flag: hay datos nuevos listos para publicar por MQTT
bool usandoWiFi      = false; // Flag: indica si se está usando Wi-Fi o GPRS

// ── Callback ESP-NOW: recepción de datos del receptor ────────
// Se ejecuta automáticamente cuando llega un mensaje ESP-NOW.
// Copia los bytes recibidos a un String y activa el flag de datos nuevos.
void OnDataRecv(const esp_now_recv_info_t *recv_info, const uint8_t *data, int len) {
  char buffer[len + 1];
  memcpy(buffer, data, len); // Copia los bytes del mensaje
  buffer[len] = '\0';        // Termina el string con null
  ultimoJSON     = String(buffer);
  hayDatosNuevos = true;
  Serial.println("[NOW] JSON recibido: " + ultimoJSON);
}

// ── Verificación de conectividad con el broker ───────────────
// Intenta abrir una conexión TCP al broker en el puerto 8883.
// Útil para diagnosticar si el puerto está bloqueado por el ISP o firewall.
bool puertoBrokerAlcanzable() {
  WiFiClientSecure testClient;
  testClient.setInsecure(); // Sin verificación de certificado (solo para el test)
  testClient.setTimeout(5000);
  bool ok = testClient.connect(broker, puerto);
  if (ok) {
    Serial.println("[NET] Puerto 8883 alcanzable");
    testClient.stop();
  } else {
    Serial.println("[NET] Puerto 8883 BLOQUEADO o broker caido");
  }
  return ok;
}

// ── Inicialización del módem SIM7000 ────────────────────────
// Secuencia de encendido por pulso en PWR_PIN, inicialización
// del UART, reinicio del módem y conexión a la red GPRS.
void iniciarModem() {
  Serial.println("[MODEM] Encendiendo SIM7000...");
  pinMode(PWR_PIN, OUTPUT);
  digitalWrite(PWR_PIN, LOW);  delay(100);
  digitalWrite(PWR_PIN, HIGH); delay(1000); // Pulso de encendido
  digitalWrite(PWR_PIN, LOW);
  ModemSerial.begin(115200, SERIAL_8N1, PIN_RX, PIN_TX);
  delay(3000); // Espera a que el módem arranque
  modem.restart();
  // Espera señal de red celular (hasta 30 segundos)
  if (!modem.waitForNetwork(30000L)) {
    Serial.println("[MODEM] Sin señal de red.");
    return;
  }
  // Conecta al APN de datos de Telcel
  if (!modem.gprsConnect(apn, "", "")) {
    Serial.println("[MODEM] Fallo GPRS.");
    return;
  }
  Serial.println("[MODEM] GPRS conectado.");
}

// ── Conexión al broker MQTT ──────────────────────────────────
// Intenta conectarse hasta MAX_INTENTOS veces con un ID aleatorio
// (evita conflictos si hay otra instancia del mismo cliente conectada).
// Imprime el código de error específico para facilitar el diagnóstico.
void conectarMQTT() {
  int intentos           = 0;
  const int MAX_INTENTOS = 5;

  while (!mqtt.connected() && intentos < MAX_INTENTOS) {
    // ID único por sesión para evitar "Session already exists" en HiveMQ
    String id = "Vetsense-ITSOEH-" + String(random(10000, 99999));
    Serial.print("[MQTT] Intentando (ID: " + id + ")... ");

    if (mqtt.connect(id.c_str(), mqtt_user, mqtt_pass)) {
      Serial.println("CONECTADO!");
    } else {
      int rc = mqtt.state(); // Código de error de PubSubClient
      Serial.println("Fallo RC=" + String(rc));
      // Decodifica el código para facilitar diagnóstico
      switch (rc) {
        case -4: Serial.println("  >> TIMEOUT");                      break;
        case -3: Serial.println("  >> Broker inalcanzable");          break;
        case -2: Serial.println("  >> Fallo de red");                 break;
        case  3: Serial.println("  >> Broker no disponible");         break;
        case  4: Serial.println("  >> Usuario/password incorrectos"); break;
        case  5: Serial.println("  >> No autorizado");                break;
        default: Serial.println("  >> Error desconocido");            break;
      }
      intentos++;
      delay(5000); // Espera 5 s antes del siguiente intento
    }
  }

  if (!mqtt.connected()) {
    Serial.println("[MQTT] No se pudo conectar tras " + String(MAX_INTENTOS) + " intentos.");
  }
}

// ── Setup ────────────────────────────────────────────────────
void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("\n===== VetSense Gateway =====");

  // ── Intentar conexión Wi-Fi primero ─────────────────────
  // Se usa modo AP_STA para que ESP-NOW y Wi-Fi coexistan
  WiFi.mode(WIFI_AP_STA);
  WiFi.begin(ssid_wifi, pass_wifi);
  Serial.print("[WIFI] Buscando red");

  int intentosWiFi = 0;
  while (WiFi.status() != WL_CONNECTED && intentosWiFi < 20) {
    delay(500);
    Serial.print(".");
    intentosWiFi++; // Máximo 10 segundos de espera
  }
  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("[WIFI] Conectado. IP: " + WiFi.localIP().toString());
    // Imprime canal y MAC para que el receptor pueda sincronizarse por ESP-NOW
    Serial.println("Canal WiFi: " + String(WiFi.channel()));
    Serial.println("MAC Gateway: " + WiFi.macAddress());
    puertoBrokerAlcanzable(); // Verifica si el puerto 8883 está abierto
    wifiClient.setInsecure(); // Acepta cualquier certificado TLS (sin validación CA)
    wifiClient.setTimeout(30000);
    mqtt.setClient(wifiClient); // MQTT usará el cliente Wi-Fi TLS
    usandoWiFi = true;
  } else {
    // Si no hay Wi-Fi disponible, cae a la conexión celular GPRS
    Serial.println("[WIFI] No encontrado. Iniciando MODEM...");
    iniciarModem();
    mqtt.setClient(gsmClient); // MQTT usará el cliente GSM
    usandoWiFi = false;
  }

  // ── Configurar y conectar MQTT ───────────────────────────
  mqtt.setBufferSize(512);          // Buffer suficiente para el JSON de 6 campos
  mqtt.setServer(broker, puerto);   // Broker HiveMQ Cloud puerto TLS 8883
  mqtt.setKeepAlive(60);            // Ping keepalive cada 60 s para mantener la sesión
  mqtt.setSocketTimeout(30);        // Timeout de socket 30 s
  conectarMQTT();

  // ── Iniciar ESP-NOW ──────────────────────────────────────
  // Registra el callback que recibe los JSON del nodo receptor
  if (esp_now_init() == ESP_OK) {
    esp_now_register_recv_cb(OnDataRecv);
    Serial.println("[NOW] ESP-NOW listo");
  } else {
    Serial.println("[NOW] Error al iniciar ESP-NOW");
  }

  Serial.println("============================\n");
}

// ── Loop ─────────────────────────────────────────────────────
void loop() {
  // ── Fallback: si se pierde Wi-Fi, cambia a GPRS ─────────
  if (usandoWiFi && WiFi.status() != WL_CONNECTED) {
    Serial.println("[WIFI] Perdido. Cambiando a GPRS...");
    iniciarModem();
    mqtt.setClient(gsmClient);
    usandoWiFi = false;
  }

  // ── Reconexión automática MQTT ───────────────────────────
  // Si se pierde la sesión MQTT (timeout, broker reiniciado, etc.)
  // intenta reconectarse antes de publicar
  if (!mqtt.connected()) {
    conectarMQTT();
  }

  // ── Publicar datos nuevos al broker ─────────────────────
  // Solo publica si llegó un JSON nuevo por ESP-NOW y hay conexión MQTT activa
  if (hayDatosNuevos && mqtt.connected()) {
    if (mqtt.publish(topic, ultimoJSON.c_str())) {
      Serial.println("[NUBE] Publicado: " + ultimoJSON);
    } else {
      Serial.println("[NUBE] Fallo al publicar.");
    }
    hayDatosNuevos = false; // Resetea el flag hasta el próximo paquete ESP-NOW
  }

  mqtt.loop(); // Mantiene viva la sesión MQTT y procesa mensajes entrantes
}