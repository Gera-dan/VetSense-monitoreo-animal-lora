// ============================================================
//  NODO RECEPTOR — Sistema de Monitoreo Animal Inalámbrico
//  Identificador: CENTRAL_RECEPTORA
//  Tarjeta: Heltec LoRa V3 (ESP32-S3 + SX1262)
//  Descripción: Escucha continuamente el canal LoRa esperando
//               paquetes del transmisor. Al recibirlos, los
//               parsea, muestra en OLED y reenvía al gateway
//               LILYGO via ESP-NOW en formato JSON.
// ============================================================

// ── Librerías ───────────────────────────────────────────────
#include <RadioLib.h>       // Control del transceptor SX1262 (LoRa)
#include <Wire.h>           // Bus I2C (pantalla OLED)
#include "SSD1306Wire.h"    // Pantalla OLED 128x64 de Heltec
#include <esp_now.h>        // Protocolo ESP-NOW para envío local al gateway
#include <WiFi.h>           // Necesario para inicializar ESP-NOW
#include "esp_wifi.h"       // Control de canal Wi-Fi (sincronización con gateway)

const String MI_NOMBRE = "CENTRAL_RECEPTORA"; // Identificador de este nodo

// ── MAC del gateway LILYGO T-SIM ────────────────────────────
// ESP-NOW necesita la dirección MAC exacta del dispositivo destino.
// Esta MAC debe coincidir con la que reporta el LILYGO al arrancar.
uint8_t direccionDestino[] = {0xC8, 0x2E, 0x18, 0xAC, 0x50, 0x90};

// ── Instancias de hardware ───────────────────────────────────
// SX1262: NSS=8, DIO1=14, RST=12, BUSY=13
SX1262 radio = new Module(8, 14, 12, 13);
// OLED SSD1306: I2C dirección 0x3C, SDA=GPIO17, SCL=GPIO18
SSD1306Wire display(0x3c, 17, 18);

// ── Pines de control de la pantalla OLED ────────────────────
#define VEXT_CONTROL 36  // LOW = enciende la alimentación de la OLED
#define RST_OLED     21  // Reset del controlador SSD1306

// ── Variables globales del último paquete recibido ───────────
float tempSujeto   = 0;  // Temperatura zona de lesión (°C)
float tempAmbiente = 0;  // Temperatura ambiente (°C)
float sensacion    = 0;  // Sensación térmica calculada (°C)
float humedad      = 0;  // Humedad relativa (%)
int   bpm          = 0;  // Frecuencia cardíaca (lpm)
int   paquete      = 0;  // Número de paquete recibido
bool  enviadoOk    = false; // Estado del último envío ESP-NOW

// ── Callback ESP-NOW: resultado del envío ───────────────────
// Se ejecuta automáticamente después de cada esp_now_send().
// Actualiza enviadoOk para mostrarlo en pantalla ("NOW" o "!!!").
void OnDataSent(const esp_now_send_info_t *info, esp_now_send_status_t status) {
  enviadoOk = (status == ESP_NOW_SEND_SUCCESS);
}

// ── Actualización de pantalla OLED ──────────────────────────
// Dos modos:
//   esperando=true  → pantalla de espera ("Buscando LoRa...")
//   esperando=false → muestra datos del último paquete recibido
void actualizarPantalla(bool esperando) {
  display.clear();
  if (esperando) {
    // Pantalla inicial mientras no llega ningún paquete
    display.setFont(ArialMT_Plain_16);
    display.drawString(0, 0, "Esperando...");
    display.setFont(ArialMT_Plain_10);
    display.drawString(0, 20, "Buscando LoRa...");
  } else {
    // Pantalla de datos: BPM grande en el centro
    display.setFont(ArialMT_Plain_24);
    String bpmStr = String(bpm) + " BPM";
    display.drawString(64 - display.getStringWidth(bpmStr)/2, 16, bpmStr); // Centrado
    display.drawLine(0, 44, 127, 44); // Línea separadora
    display.setFont(ArialMT_Plain_10);
    // Fila inferior: temperatura del sujeto | humedad | número de paquete
    display.drawString(0,  47, String(tempSujeto, 1) + "C");
    display.drawString(45, 47, String(humedad, 0) + "%");
    display.drawString(90, 47, "Pkt:" + String(paquete));
    // Fila superior: confirmación de recepción y estado ESP-NOW
    display.drawString(0,  0,  "RX OK #" + String(paquete));
    display.drawString(105, 0, enviadoOk ? "NOW" : "!!!"); // NOW=enviado OK, !!!=fallo
  }
  display.display(); // Vuelca el buffer a la pantalla física
}

// ── Parser del paquete LoRa ──────────────────────────────────
// Extrae cada campo del formato compacto recibido por LoRa:
// "T:36.5|A:24.1|S:25.3|H:60.0|B:75|P:12"
// Usa indexOf() para localizar cada clave y substring() para extraer el valor.
void parsearPaquete(String datos) {
  int idx, end;

  // T: Temperatura del sujeto (DS18B20 bajo vendaje)
  idx = datos.indexOf("T:") + 2;
  end = datos.indexOf("|", idx);
  tempSujeto = datos.substring(idx, end).toFloat();

  // A: Temperatura ambiente (DHT11)
  idx = datos.indexOf("|A:") + 3;
  end = datos.indexOf("|", idx);
  tempAmbiente = datos.substring(idx, end).toFloat();

  // S: Sensación térmica (Heat Index calculado en el transmisor)
  idx = datos.indexOf("|S:") + 3;
  end = datos.indexOf("|", idx);
  sensacion = datos.substring(idx, end).toFloat();

  // H: Humedad relativa (DHT11)
  idx = datos.indexOf("|H:") + 3;
  end = datos.indexOf("|", idx);
  humedad = datos.substring(idx, end).toFloat();

  // B: Frecuencia cardíaca en BPM (AD8232)
  idx = datos.indexOf("|B:") + 3;
  end = datos.indexOf("|", idx);
  bpm = datos.substring(idx, end).toInt();

  // P: Número de paquete (para detectar paquetes perdidos)
  idx = datos.indexOf("|P:") + 3;
  paquete = datos.substring(idx).toInt(); // Último campo: no necesita 'end'
}

// ── Constructor del JSON para el gateway ────────────────────
// Convierte las variables globales a un JSON con claves descriptivas
// que el LILYGO publicará directamente al broker MQTT HiveMQ.
String construirJSON() {
  return "{"
    "\"Temperatura_Sujeto\":"   + String(tempSujeto,   1) + ","
    "\"Temperatura_Ambiente\":" + String(tempAmbiente, 1) + ","
    "\"Sensacion_Termica\":"    + String(sensacion,    1) + ","
    "\"Humedad\":"              + String(humedad,      1) + ","
    "\"Frecuencia_Cardiaca\":"  + String(bpm)             + ","
    "\"Paquete\":"              + String(paquete)          +
  "}";
}

// ── Setup ────────────────────────────────────────────────────
void setup() {
  Serial.begin(115200);
  delay(1000);

  // ── Configurar canal Wi-Fi para ESP-NOW ─────────────────
  // ESP-NOW usa el stack Wi-Fi internamente. Ambos dispositivos
  // (receptor y gateway LILYGO) deben estar en el mismo canal.
  // Se fija en canal 6 para evitar interferencia con redes Wi-Fi externas.
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  delay(100);
  esp_wifi_set_channel(6, WIFI_SECOND_CHAN_NONE);

  // ── Encender pantalla OLED ───────────────────────────────
  pinMode(VEXT_CONTROL, OUTPUT);
  digitalWrite(VEXT_CONTROL, LOW); // LOW activa el regulador de la OLED
  delay(500);
  pinMode(RST_OLED, OUTPUT);
  digitalWrite(RST_OLED, LOW); delay(50);
  digitalWrite(RST_OLED, HIGH);
  display.init();
  display.flipScreenVertically();
  display.clear();
  display.display();

  // ── Iniciar ESP-NOW ──────────────────────────────────────
  if (esp_now_init() != ESP_OK) {
    Serial.println("Error inicializando ESP-NOW");
    return; // Sin ESP-NOW no puede reenviar datos al gateway
  }
  // Registra el callback que se llama tras cada envío
  esp_now_register_send_cb(OnDataSent);

  // ── Registrar el gateway LILYGO como peer ESP-NOW ────────
  // Se debe registrar el dispositivo destino antes de poder enviarle datos.
  esp_now_peer_info_t peerInfo = {};
  memcpy(peerInfo.peer_addr, direccionDestino, 6); // Copia los 6 bytes de la MAC
  peerInfo.channel = 6;       // Mismo canal que el configurado arriba
  peerInfo.encrypt = false;   // Sin cifrado (simplifica la configuración)
  peerInfo.ifidx   = WIFI_IF_STA;
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("No se pudo agregar peer");
  }

  // ── Iniciar radio LoRa SX1262 ────────────────────────────
  // Parámetros idénticos al transmisor para que se puedan comunicar
  SPI.begin(9, 11, 10, 8); // SCK, MISO, MOSI, NSS
  int state = radio.begin(433.0);       // Frecuencia 433 MHz
  radio.setDio2AsRfSwitch(true);        // DIO2 controla el switch RF de la Heltec
  radio.setSpreadingFactor(12);         // SF12: debe coincidir con el transmisor
  radio.setBandwidth(62.5);             // BW 62.5 kHz: debe coincidir con el transmisor
  radio.setCodingRate(8);               // CR 4/8: debe coincidir con el transmisor
  radio.setOutputPower(22);             // Potencia (no relevante en RX, pero se configura)
  radio.setCurrentLimit(140.0);         // Límite de corriente del regulador
  radio.setSyncWord(0xAB);              // SyncWord: filtra paquetes de otras redes LoRa
  if (state == RADIOLIB_ERR_NONE) {
    Serial.println("LoRa OK");
  } else {
    Serial.print("Fallo LoRa, codigo: ");
    Serial.println(state); // Código de error de RadioLib para diagnóstico
  }

  // Muestra pantalla de espera hasta recibir el primer paquete
  actualizarPantalla(true);
}

// ── Loop ─────────────────────────────────────────────────────
// El receptor opera en modo de escucha continua (blocking receive).
// Cada iteración intenta recibir un paquete LoRa.
// Si llega uno válido: parsea → actualiza pantalla → reenvía por ESP-NOW.
void loop() {
  String datos = "";
  // radio.receive() bloquea hasta recibir un paquete o detectar error.
  // Devuelve RADIOLIB_ERR_NONE si el paquete pasó el CRC correctamente.
  int state = radio.receive(datos);

  if (state == RADIOLIB_ERR_NONE) {
    Serial.println("LoRa RX: " + datos); // Log del paquete crudo para depuración

    parsearPaquete(datos);      // Extrae los 6 campos del payload compacto
    actualizarPantalla(false);  // Muestra los datos en la OLED

    // Construye el JSON y lo envía al gateway LILYGO por ESP-NOW
    String json = construirJSON();
    esp_err_t result = esp_now_send(
      direccionDestino,           // MAC del LILYGO
      (uint8_t *)json.c_str(),    // Datos como array de bytes
      json.length()               // Longitud en bytes
    );
    if (result != ESP_OK) {
      Serial.println("ESP-NOW error al enviar");
    }
    // El resultado real del envío llega de forma asíncrona en OnDataSent()
  }
  // Si state != RADIOLIB_ERR_NONE es ruido, paquete corrupto o CRC fallido;
  // se ignora y se vuelve a escuchar en la siguiente iteración.

  delay(10); // Pequeña pausa para no saturar el CPU
}