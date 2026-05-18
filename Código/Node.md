# Node-Red
![DIAGRAMA](.../img/Node-red.png)

# Sistema de Telemetría Animal — Documentación Técnica

## Arquitectura de Red e Ingesta de Datos

El diseño del sistema asegura una alta disponibilidad y tolerancia a fallos en la transmisión de datos desde el campo hasta los sistemas de almacenamiento:

| # | Componente | Descripción |
|---|------------|-------------|
| 1 | **Dispositivos Embebidos (Nodos Sensores)** | Los collares o dispositivos instalados en los animales recolectan las variables y las publican mediante el protocolo industrial MQTT. |
| 2 | **Broker en la Nube (HiveMQ)** | Actúa como el primer punto de recepción a nivel global, gestionando la llegada masiva de paquetes. |
| 3 | **Broker MQTT Privado** | Se establece un puente de red (*bridge*) que redirige el tráfico desde HiveMQ hacia un servidor MQTT privado para garantizar un mayor control, aislamiento y seguridad de la información. |
| 4 | **Captura (Node-RED)** | El flujo inicia con el nodo morado `vetsense/datos/animal`, el cual mantiene una conexión persistente con el broker privado y se activa inmediatamente cada vez que un dispositivo publica un nuevo paquete de telemetría. |

---

## Componentes del Flujo y Lógica de Programación

### 1. Preprocesamiento de Datos — Nodo JSON

Los paquetes de datos que transitan por la red MQTT son recibidos en formato de texto plano (`string`). Para que Node-RED pueda interpretar las variables, este nodo analiza el texto entrante y lo convierte en un objeto JSON nativo de JavaScript.

Esto permite que los nodos subsecuentes accedan a las propiedades individuales (como temperatura o pulso) mediante la sintaxis de objetos:

```
msg.payload.Variable
```

---

### 2. Procesamiento para Visualización y Dashboard — Nodo `function 4`

Este nodo recibe el objeto JSON procesado y actúa como un **distribuidor inteligente**. Su función principal es:

- ✅ Validar la consistencia de los datos
- 🚫 Filtrar lecturas erróneas causadas por fallos de hardware
- 📤 Segmentar el paquete en **7 salidas independientes** dirigidas a los componentes de la interfaz gráfica

#### Salidas del nodo

| Salida | Topic | Variable |
|--------|-------|----------|
| 1 | `Frecuencia_Cardiaca` | BPM (valor flotante) |
| 2 | `Pulso` | BPM (redondeado) |
| 3 | `Temperatura_Sujeto` | Temperatura del animal (°C) |
| 4 | `Temperatura_Ambiente` | Temperatura del entorno (°C) |
| 5 | `Sensacion_Termica` | Sensación térmica calculada |
| 6 | `Humedad` | Humedad relativa (%) |
| 7 | `Paquete` | Número de paquete |

#### Código de la Función

```javascript
// Asegurarse que el payload sea objeto
if (typeof msg.payload === "string") {
    msg.payload = JSON.parse(msg.payload);
}

// 1. Extraer datos
let bpm         = msg.payload.Frecuencia_Cardiaca   || 0;
let tempSujeto  = msg.payload.Temperatura_Sujeto    || 0;
let tempAmbiente= msg.payload.Temperatura_Ambiente  || 0;
let sensacion   = msg.payload.Sensacion_Termica     || 0;
let hum         = msg.payload.Humedad               || 0;
let paquete     = msg.payload.Paquete               || 0;

// 2. Manejo de errores de hardware
if (tempSujeto   <= -127) tempSujeto   = 0;
if (tempAmbiente <= -127) tempAmbiente = 0;

// 3. Distribución a las terminales de salida
return [
    { payload: bpm,                              topic: "Frecuencia_Cardiaca" },
    { payload: Math.round(bpm),                  topic: "Pulso"               },
    { payload: tempSujeto,                       topic: "Temperatura_Sujeto"  },
    { payload: tempAmbiente,                     topic: "Temperatura_Ambiente"},
    { payload: sensacion,                        topic: "Sensacion_Termica"   },
    { payload: isNaN(hum) ? 0 : Math.round(hum), topic: "Humedad"             },
    { payload: paquete,                          topic: "Paquete"             }
];
```

> **Nota sobre el manejo de errores:** Los sensores de temperatura DS18B20 devuelven `-127` cuando ocurre un fallo de lectura o desconexión. El nodo detecta este valor y lo reemplaza por `0` para evitar que datos corruptos lleguen al dashboard.

