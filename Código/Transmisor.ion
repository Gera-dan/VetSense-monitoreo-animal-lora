// ============================================================
//  NODO TRANSMISOR — Sistema de Monitoreo Animal Inalámbrico
//  Identificador: NODO_MEDICO_01
//  Tarjeta: Heltec LoRa V3 (ESP32-S3 + SX1262)
//  Comunicación: LoRa 433 MHz + BLE (app NodoVet)
//  Descripción: Lee sensores biométricos y ambientales del animal,
//               calcula BPM con filtros, sensación térmica, y
//               transmite los datos por LoRa cada 10 s.
//               Entre transmisiones entra en deep-sleep 60 s
//               para ahorrar energía de la batería LiPo.
// ============================================================

// ── Librerías ───────────────────────────────────────────────
#include <RadioLib.h>         // Control del transceptor SX1262 (LoRa)
#include <Wire.h>             // Bus I2C (pantalla OLED)
#include "SSD1306Wire.h"      // Pantalla OLED 128x64 de Heltec
#include <OneWire.h>          // Protocolo 1-Wire (DS18B20)
#include <DallasTemperature.h>// Sensor de temperatura DS18B20
#include <DHT.h>              // Sensores DHT11 (humedad y temp. ambiente)
#include <math.h>             // Funciones matemáticas (exp, fórmula Heat Index)
#include <BLEDevice.h>        // BLE: inicialización del dispositivo
#include <BLEServer.h>        // BLE: servidor GATT
#include <BLEUtils.h>         // BLE: utilidades
#include <BLE2902.h>          // BLE: descriptor para notificaciones
#include "esp_sleep.h"        // Control de deep-sleep del ESP32-S3

// ── Constantes de configuración ─────────────────────────────
#define SAMPLES               20       // Nº de muestras para el filtro de media móvil del AD8232
#define NUM_MEDICIONES         3       // Nº de lecturas BPM para el filtro de mediana
#define CALIBRATION_TIME   15000      // Duración de calibración del AD8232 al inicio (ms)
#define MEASUREMENT_INTERVAL 10000    // Duración de la ventana de medición de BPM (ms)
#define MAX_VARIACION_BPM     25      // Máxima variación aceptada entre lecturas de BPM (lpm)
#define SLEEP_DURATION_US 60000000ULL // Duración del deep-sleep: 60 segundos en microsegundos

// ── Identificadores del nodo ─────────────────────────────────
const String MI_NOMBRE = "NODO_MEDICO_01";    // Nombre de este nodo transmisor
const String DESTINO   = "CENTRAL_RECEPTORA"; // Nombre del nodo receptor esperado

// ── UUIDs del servicio BLE ───────────────────────────────────
// Identifican el servicio y características GATT que la app
// NodoVet usa para conectarse, configurar el animal y recibir datos
#define SERVICE_UUID  "12345678-1234-1234-1234-123456789abc" // Servicio principal
#define CHAR_CONFIG   "12345678-1234-1234-1234-123456789ab1" // Escritura: configurar animal
#define CHAR_STATUS   "12345678-1234-1234-1234-123456789ab2" // Notificación: enviar datos a la app

// ── Perfiles de animales ─────────────────────────────────────
// Define el rango fisiológico válido de BPM por especie.
// El BPM calculado se valida contra este rango antes de aceptarse.
struct AnimalConfig {
  const char* nombre;
  int bpmMin;
  int bpmMax;
};
AnimalConfig animales[] = {
  { "Perro_Grande",   60, 130 },  // Índice 0
  { "Perro_Pequeno", 100, 180 },  // Índice 1
  { "Borrego",        60, 120 },  // Índice 2
  { "Vaca",           40,  80 },  // Índice 3
};

// ── Variables en memoria RTC ─────────────────────────────────
// RTC_DATA_ATTR: persisten en la memoria RTC del ESP32-S3 durante
// el deep-sleep, por lo que el nodo "recuerda" su estado entre ciclos.
RTC_DATA_ATTR int   animalSeleccionado = 0;     // Perfil animal activo (configurable por BLE)
RTC_DATA_ATTR int   ContadorPaquetes   = 0;     // Contador acumulado de paquetes enviados
RTC_DATA_ATTR int   bpmAnterior        = 0;     // Último BPM válido (filtro de variación)
RTC_DATA_ATTR float mediciones[NUM_MEDICIONES]; // Historial BPM para filtro de mediana
RTC_DATA_ATTR int   medicionIndex      = 0;     // Índice circular del historial
RTC_DATA_ATTR bool  calibrado          = false; // Flag: ¿ya se calibró el AD8232?
RTC_DATA_ATTR int   umbralAlto         = 25;    // Umbral de detección de pico del AD8232
RTC_DATA_ATTR int   maxVal             = 0;     // Máximo registrado en calibración
RTC_DATA_ATTR int   minVal             = 4095;  // Mínimo registrado en calibración

// ── Instancias de hardware ───────────────────────────────────
// SX1262: NSS=8, DIO1=14, RST=12, BUSY=13
SX1262 radio = new Module(8, 14, 12, 13);
// OLED SSD1306: dirección I2C 0x3C, SDA=GPIO17, SCL=GPIO18
SSD1306Wire display(0x3c, 17, 18);

// ── Pines ────────────────────────────────────────────────────
#define DS18B20_PIN     3   // Temperatura lesión 1-Wire. Pull-up 4.7kΩ a 3.3V
#define DHTPIN_HUMEDAD  34  // DHT11 dedicado solo a humedad relativa
#define DHTPIN_TEMP     1   // DHT11 dedicado solo a temperatura ambiente
#define AD8232_OUT      6   // Salida analógica ECG del AD8232
#define AD8232_LO_PLUS  5   // Detección electrodo suelto LO+
#define AD8232_LO_MINUS 7   // Detección electrodo suelto LO-
#define VEXT_CONTROL    36  // LOW = enciende alimentación de la OLED
#define RST_OLED        21  // Reset del controlador SSD1306

// ── Sensores ─────────────────────────────────────────────────
#define DHTTYPE DHT11
DHT dhtHumedad(DHTPIN_HUMEDAD, DHTTYPE); // Instancia DHT11 para humedad
DHT dhtTemp(DHTPIN_TEMP, DHTTYPE);       // Instancia DHT11 para temperatura ambiente
OneWire oneWire(DS18B20_PIN);            // Bus 1-Wire
DallasTemperature sensorTemp(&oneWire);  // Librería DS18B20

// ── Variables del algoritmo de BPM ──────────────────────────
int readings[SAMPLES];          // Buffer circular del filtro de media móvil
int readIndex     = 0;          // Índice actual del buffer
long total        = 0;          // Suma acumulada (evita recalcular el promedio completo)
unsigned long lastPeakTime = 0; // Tiempo del último pico detectado (ms)
int beatCount     = 0;          // Latidos contados en la ventana de medición
int bpmFinal      = 0;          // BPM final filtrado y validado
bool peakDetected = false;      // Anti-rebote: bloquea doble detección del mismo pico

// ── Variables ambientales ────────────────────────────────────
float humedad      = 0; // Humedad relativa (%) del DHT11 en GPIO 34
float tempAmbiente = 0; // Temperatura ambiente (°C) del DHT11 en GPIO 1

// ── BLE ──────────────────────────────────────────────────────
BLEServer*         pServer      = nullptr;
BLECharacteristic* pCharStatus  = nullptr;
bool               bleConectado = false;

// ── Callbacks BLE: conexión y desconexión ────────────────────
class ServerCallbacks : public BLEServerCallbacks {
  // Se llama cuando la app NodoVet se conecta por BLE
  void onConnect(BLEServer* pServer) override {
    bleConectado = true;
    Serial.println("BLE: cliente conectado");
  }
  // Se llama cuando la app se desconecta; reinicia el advertising
  void onDisconnect(BLEServer* pServer) override {
    bleConectado = false;
    BLEDevice::startAdvertising(); // Vuelve a anunciarse para nuevas conexiones
  }
};

// ── Callback BLE: configuración del animal ───────────────────
// La app escribe "ANIMAL:X" donde X es el índice (0–3) del perfil.
// Se guarda en RTC para que persista entre ciclos de deep-sleep.
class ConfigCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* pChar) override {
    String val = pChar->getValue().c_str();
    if (val.startsWith("ANIMAL:")) {
      int idx = val.substring(7).toInt();
      if (idx >= 0 && idx <= 3) {
        animalSeleccionado = idx;
      }
    }
  }
};

// ── Inicialización del servidor BLE ─────────────────────────
void iniciarBLE() {
  BLEDevice::init("NodoMedico_01"); // Nombre visible en el escaneo Bluetooth
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new ServerCallbacks());

  BLEService* pService = pServer->createService(SERVICE_UUID);

  // Característica de escritura: la app configura el tipo de animal
  BLECharacteristic* pCharConfig = pService->createCharacteristic(
    CHAR_CONFIG, BLECharacteristic::PROPERTY_WRITE
  );
  pCharConfig->setCallbacks(new ConfigCallbacks());

  // Característica de notificación: el nodo envía datos a la app
  pCharStatus = pService->createCharacteristic(
    CHAR_STATUS, BLECharacteristic::PROPERTY_NOTIFY
  );
  pCharStatus->addDescriptor(new BLE2902()); // Habilita notificaciones en el cliente

  pService->start();
  BLEAdvertising* pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(true);
  BLEDevice::startAdvertising();
}

// ── Envío de notificación BLE ────────────────────────────────
// Solo envía si hay una app conectada; el delay(100) da tiempo
// al stack BLE para procesar la notificación antes del deep-sleep
void enviarBLE(String datos) {
  if (bleConectado && pCharStatus != nullptr) {
    pCharStatus->setValue(datos.c_str());
    pCharStatus->notify();
    delay(100);
  }
}

// ── Cálculo de sensación térmica ─────────────────────────────
// Tres fórmulas según rango de temperatura y humedad:
//   temp < 27°C                    → Steadman simplificada
//   temp >= 27°C y humedad < 40%   → Magnus con vapor de agua
//   temp >= 27°C y humedad >= 40%  → Heat Index completo de Rothfusz
float calcularSensacionTermica(float tempC, float humedad) {
  if (tempC < 27.0) {
    return tempC - 0.4 * (tempC - 10.0) * (1.0 - humedad / 100.0);
  }
  if (humedad < 40.0) {
    return tempC + (0.33 * (humedad / 100.0 * 6.105 *
           exp(17.27 * tempC / (237.7 + tempC)))) - 4.0;
  }
  // Rothfusz opera en °F; se convierte entrada y salida
  float T  = tempC * 9.0 / 5.0 + 32.0;
  float R  = humedad;
  float HI = -42.379
    + 2.04901523  * T
    + 10.14333127 * R
    - 0.22475541  * T * R
    - 0.00683783  * T * T
    - 0.05481717  * R * R
    + 0.00122874  * T * T * R
    + 0.00085282  * T * R * R
    - 0.00000199  * T * T * R * R;
  return (HI - 32.0) * 5.0 / 9.0; // Regresa a °C
}

// ── Mediana de BPM ───────────────────────────────────────────
// Ordena las últimas NUM_MEDICIONES lecturas válidas y devuelve
// el valor central. Más robusta que el promedio ante valores atípicos.
float calcularMediana() {
  float temp[NUM_MEDICIONES];
  int count = 0;
  for (int i = 0; i < NUM_MEDICIONES; i++)
    if (mediciones[i] > 0) temp[count++] = mediciones[i];
  // Ordenamiento burbuja (NUM_MEDICIONES=3, costo insignificante)
  for (int i = 0; i < count - 1; i++)
    for (int j = i + 1; j < count; j++)
      if (temp[j] < temp[i]) { float a=temp[i]; temp[i]=temp[j]; temp[j]=a; }
  return count > 0 ? temp[count / 2] : 0;
}

// ── Pantalla OLED ────────────────────────────────────────────
// BPM grande centrado, línea separadora, y en la fila inferior:
// temperatura lesión | humedad | contador de paquetes.
// Fila superior: estado de envío y si hay BLE conectado.
void actualizarPantalla(float temp, float humedad, int bpm, bool enviando) {
  display.clear();
  display.setFont(ArialMT_Plain_24);
  String bpmStr = String(bpm) + " BPM";
  display.drawString(64 - display.getStringWidth(bpmStr)/2, 18, bpmStr);
  display.drawLine(0, 44, 127, 44);
  display.setFont(ArialMT_Plain_10);
  display.drawString(0,  47, String(temp, 1) + "C");
  display.drawString(45, 47, String(humedad, 0) + "%");
  display.drawString(90, 47, "Pkt:" + String(ContadorPaquetes));
  String estado = enviando ? "Enviando..." : "OK #" + String(ContadorPaquetes);
  if (bleConectado) estado += " BLE";
  display.drawString(0, 0, estado);
  display.display();
}

// ── Lectura DHT11 humedad ────────────────────────────────────
// Reintenta una vez con delay(500) si la primera lectura da NaN
void leerDHT11Humedad() {
  float h = dhtHumedad.readHumidity();
  if (!isnan(h)) { humedad = h; return; }
  delay(500);
  h = dhtHumedad.readHumidity();
  if (!isnan(h)) humedad = h;
}

// ── Lectura DHT11 temperatura ambiente ──────────────────────
void leerDHT11Temperatura() {
  float t = dhtTemp.readTemperature();
  if (!isnan(t)) { tempAmbiente = t; return; }
  delay(500);
  t = dhtTemp.readTemperature();
  if (!isnan(t)) tempAmbiente = t;
}

// ============================================================
//  SETUP — Se ejecuta en cada despertar del deep-sleep.
//  Toda la lógica está aquí; loop() queda vacío porque al
//  terminar setup() el nodo vuelve a dormir.
// ============================================================
void setup() {
  Serial.begin(115200);

  // ── Encender OLED ────────────────────────────────────────
  pinMode(VEXT_CONTROL, OUTPUT);
  digitalWrite(VEXT_CONTROL, LOW); // LOW activa el regulador de la OLED
  delay(500);
  pinMode(RST_OLED, OUTPUT);
  digitalWrite(RST_OLED, LOW); delay(50);
  digitalWrite(RST_OLED, HIGH);
  display.init();
  display.flipScreenVertically();
  display.clear();

  // ── Pines del AD8232 ─────────────────────────────────────
  // HIGH en LO+ o LO- indica que un electrodo está desconectado
  pinMode(AD8232_LO_PLUS,  INPUT);
  pinMode(AD8232_LO_MINUS, INPUT);

  // ── Iniciar sensores ─────────────────────────────────────
  dhtHumedad.begin();
  dhtTemp.begin();
  delay(1000);        // DHT11 requiere ≥1 s tras encendido
  sensorTemp.begin(); // Escanea el bus 1-Wire y detecta el DS18B20

  // ── Iniciar radio LoRa ───────────────────────────────────
  SPI.begin(9, 11, 10, 8); // SCK, MISO, MOSI, NSS
  int state = radio.begin(433.0);
  radio.setDio2AsRfSwitch(true);  // DIO2 controla el switch RF de la Heltec
  radio.setSpreadingFactor(12);   // SF12: máximo alcance
  radio.setBandwidth(62.5);       // BW 62.5 kHz: +3 dB de sensibilidad
  radio.setCodingRate(8);         // CR 4/8: máxima redundancia
  radio.setOutputPower(22);       // +22 dBm: potencia máxima del SX1262
  radio.setCurrentLimit(140.0);   // Protege el regulador de la tarjeta
  radio.setSyncWord(0xAB);        // Filtra paquetes de otras redes LoRa
  if (state == RADIOLIB_ERR_NONE) Serial.println("LoRa OK");

  // ── Calibración del AD8232 (solo primer arranque) ────────
  // `calibrado` está en RTC: true si ya se calibró en un ciclo anterior.
  // El umbral calculado también persiste en RTC entre ciclos de sleep.
  if (!calibrado) {
    display.setFont(ArialMT_Plain_16);
    display.drawString(0, 0, "Calibrando...");
    display.setFont(ArialMT_Plain_10);
    display.drawString(0, 20, "Quedese quieto");
    display.drawString(0, 32, "15 segundos");
    display.display();

    unsigned long calibrationStart = millis();
    while (millis() - calibrationStart < CALIBRATION_TIME) {
      int v = analogRead(AD8232_OUT);
      if (v > maxVal) maxVal = v; // Registra el pico máximo de la señal ECG
      if (v < minVal) minVal = v; // Registra el valle mínimo
      delay(10);
    }
    // Umbral = 60% del rango pico a pico, acotado entre 30 y 80
    int rango  = maxVal - minVal;
    umbralAlto = constrain((int)(rango * 0.6), 30, 80);
    calibrado  = true;
    for (int i = 0; i < NUM_MEDICIONES; i++) mediciones[i] = -1;
  }

  // ── Iniciar BLE ──────────────────────────────────────────
  iniciarBLE();

  // ── Ventana de medición de BPM (10 segundos) ────────────
  // Lee continuamente el AD8232 y cuenta los picos (latidos)
  // durante MEASUREMENT_INTERVAL ms usando un filtro de media móvil.
  display.clear();
  display.setFont(ArialMT_Plain_10);
  display.drawString(0, 0, "Midiendo BPM...");
  display.display();

  for (int i = 0; i < SAMPLES; i++) readings[i] = 0; // Limpia el buffer

  unsigned long ventanaInicio = millis();
  while (millis() - ventanaInicio < MEASUREMENT_INTERVAL) {
    int valor = analogRead(AD8232_OUT);

    // Solo procesa si los electrodos están bien colocados
    if (!(digitalRead(AD8232_LO_PLUS) || digitalRead(AD8232_LO_MINUS))) {

      // Filtro de media móvil: elimina el componente DC del ECG
      total -= readings[readIndex];
      readings[readIndex] = valor;
      total += readings[readIndex];
      readIndex = (readIndex + 1) % SAMPLES;
      int promedio   = total / SAMPLES;
      int diferencia = valor - promedio; // Componente AC (donde están los picos)

      // Detección de pico con anti-rebote de 600 ms
      // (equivale a BPM máximo de 100, suficiente para todas las especies)
      if (diferencia > umbralAlto && !peakDetected) {
        unsigned long ahora = millis();
        if (ahora - lastPeakTime > 600) {
          peakDetected = true;
          lastPeakTime = ahora;
          beatCount++;
        }
      }
      // Resetea el flag cuando la señal baja al 50% del umbral
      if (diferencia < umbralAlto * 0.5) peakDetected = false;
    }
    delay(10); // ~100 muestras/s
  }

  // ── Cálculo del BPM final ────────────────────────────────
  // BPM = (latidos / segundos) × 60
  float heartRate = (beatCount / (MEASUREMENT_INTERVAL / 1000.0)) * 60.0;
  int bpmMin = animales[animalSeleccionado].bpmMin;
  int bpmMax = animales[animalSeleccionado].bpmMax;

  if (heartRate >= bpmMin && heartRate <= bpmMax) {
    // Dentro del rango fisiológico: aplica filtro de variación
    if (bpmAnterior == 0 || abs(heartRate - bpmAnterior) <= MAX_VARIACION_BPM) {
      // Variación aceptable: agrega al historial y recalcula mediana
      mediciones[medicionIndex % NUM_MEDICIONES] = heartRate;
      medicionIndex++;
      bpmFinal    = (int)calcularMediana();
      bpmAnterior = bpmFinal;
    } else {
      // Cambio brusco: descarta pero actualiza referencia para el siguiente ciclo
      bpmAnterior = (int)heartRate;
    }
  } else if (heartRate == 0) {
    // Sin latidos detectados: reinicia historial (electrodo suelto o animal quieto)
    for (int i = 0; i < NUM_MEDICIONES; i++) mediciones[i] = -1;
    medicionIndex = 0;
    bpmFinal = bpmAnterior = 0;
  }
  // heartRate fuera de rango pero > 0: artefacto, se conserva bpmFinal anterior

  // ── Lectura de sensores ──────────────────────────────────
  ContadorPaquetes++;
  leerDHT11Humedad();
  leerDHT11Temperatura();
  sensorTemp.requestTemperatures();
  float tempCuerpo = sensorTemp.getTempCByIndex(0); // Temperatura zona de lesión
  float sensacion  = calcularSensacionTermica(tempAmbiente, humedad);

  actualizarPantalla(tempCuerpo, humedad, bpmFinal, true);

  // ── Envío LoRa ───────────────────────────────────────────
  // Formato compacto clave:valor separado por | para minimizar el
  // Time on Air (ToA) y el consumo energético por transmisión
  String paquete = "T:" + String(tempCuerpo,   1)
    + "|A:" + String(tempAmbiente, 1)
    + "|S:" + String(sensacion,    1)
    + "|H:" + String(humedad,      1)
    + "|B:" + String(bpmFinal)
    + "|P:" + String(ContadorPaquetes);

  int stateTx = radio.transmit(paquete); // Bloqueante hasta completar la transmisión
  if (stateTx == RADIOLIB_ERR_NONE) Serial.println("LoRa OK");
  else Serial.println("Error LoRa: " + String(stateTx));

  // ── Envío BLE a la app NodoVet ───────────────────────────
  // Formato más descriptivo con nombres completos de campo
  String bleDatos = "Temperatura_Sujeto:"   + String(tempCuerpo,   1)
    + "|Temperatura_Ambiente:" + String(tempAmbiente, 1)
    + "|Sensacion_Termica:"    + String(sensacion,    1)
    + "|Humedad:"              + String(humedad,      1)
    + "|Frecuencia_Cardiaca:"  + String(bpmFinal)
    + "|Animal:"               + String(animales[animalSeleccionado].nombre)
    + "|Paquete:"              + String(ContadorPaquetes);
  enviarBLE(bleDatos);

  actualizarPantalla(tempCuerpo, humedad, bpmFinal, false);
  delay(2000); // 2 s para que el usuario lea la pantalla

  // ── Deep-sleep 60 segundos ───────────────────────────────
  // Apaga la OLED antes de dormir para ahorrar energía
  display.clear();
  display.display();
  digitalWrite(VEXT_CONTROL, HIGH); // HIGH corta la alimentación de la OLED
  Serial.println("Entrando deep-sleep 1 minuto...");
  Serial.flush();
  // Al despertar el ESP32-S3 reinicia desde setup() con las variables RTC intactas
  esp_sleep_enable_timer_wakeup(SLEEP_DURATION_US);
  esp_deep_sleep_start();
}

// ── Loop vacío ───────────────────────────────────────────────
// Nunca se ejecuta: al terminar setup() el nodo entra en deep-sleep
void loop() {}