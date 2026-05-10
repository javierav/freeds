# Arquitectura de FreeDS

> Mapa global del firmware: qué capas hay, cómo arranca el dispositivo, cómo
> se planifica el trabajo, cómo fluye un dato del inversor hasta el triac, y
> cómo se asegura la robustez.

## 1. Stack tecnológico

| Capa                 | Componente                                    |
|----------------------|-----------------------------------------------|
| Hardware             | ESP32 (Heltec WiFi Kit 32) + OLED SSD1306 + triac/dimmer + 4 relés + DS18B20 + opcional pinza SCT-013 |
| Toolchain            | PlatformIO `espressif32 ~3.5.0`, framework `arduino` |
| RTOS                 | FreeRTOS (subyacente al framework Arduino)    |
| Librerías clave      | `AsyncTCP`, `ESPAsyncWebServer`, `AsyncMqttClient`, `ArduinoJson`, `TickerScheduler`, `esp32ModbusTCP`, `OneWire`/`DallasTemperature`, `fauxmoESP`, `PID_v1`, `SSD1306` |
| Persistencia         | EEPROM emulada (`Preferences`/`EEPROM` API), SPIFFS |
| UI cliente           | Bootstrap 4 + jQuery 3 + SB-Admin-2 + canvas-gauges |
| Comunicación cliente | HTTP + Server-Sent Events (SSE) + MQTT + UPnP (Alexa) |

## 2. Estructura del repositorio

```
freeds/
├── src/                 # 17 ficheros .ino — todo el firmware (~8 KLOC)
├── include/             # cabeceras propias (modos, bitmaps, traducciones)
├── data/                # contenido de SPIFFS (HTML/CSS/JS gzipeados, idiomas)
├── languages/           # ficheros .json/.js de traducción
├── lib.zip              # librerías de terceros (descomprimir en lib/)
├── hardware/            # esquemáticos KiCad/PCB (clinxer, sanchez-rev1, KRIDA)
├── platformio.ini       # configuración de build
├── pio/name-firmware.py # script extra: nombra el binario tras compilar
├── Changelog            # histórico de versiones
└── docs/                # ESTA documentación
```

Todos los `.ino` se concatenan a un solo `sketch` por el preprocesador de
Arduino: las funciones de cualquier fichero son visibles globalmente, igual
que las variables `extern`-eable. Por eso no encontrarás `#include` cruzados
entre los `.ino` — comparten un único *translation unit*.

## 3. Diagrama de bloques (alto nivel)

```
                        ┌──────────────────────────┐
                        │    Setup() / Loop()      │
                        │  (FreeDS.ino, núcleo)    │
                        └────┬──────────────┬──────┘
                             │              │
                ┌────────────▼──┐       ┌───▼────────────────┐
                │ TickerScheduler│      │  Watchdog hw_timer │
                │  7 callbacks   │      │  30 s reset        │
                └────────┬───────┘      └────────────────────┘
                         │
   ┌─────────────────────┼──────────────────────────────────┐
   │                     │                                  │
   ▼                     ▼                                  ▼
┌──────────┐     ┌────────────────┐               ┌─────────────────┐
│ Display  │     │ Data acquisit. │ getSensorData │  Background     │
│ showOled │     │ (Support_fns)  │               │  services       │
│ Data ()  │     │                │               │                 │
└──────────┘     └───────┬────────┘               │ - WiFi reconnect│
                         │ Dispatch by config.    │ - MQTT pub/sub  │
                         │  wversion              │ - Web events    │
        ┌────────────────┼────────────────┐       │ - NTP / mDNS    │
        ▼                ▼                ▼       └─────────────────┘
┌───────────────┐ ┌────────────┐ ┌─────────────────┐
│ Serial / ESP01│ │ HTTP async │ │ Modbus RTU/TCP  │
│ (Solax v2)    │ │ (Solax v1, │ │ (DDS, SDM, SMA, │
│               │ │ Wibeee,    │ │  Victron,       │
│               │ │ Shelly,    │ │  SolarEdge…)    │
│               │ │ Fronius)   │ │                 │
└───────┬───────┘ └─────┬──────┘ └────────┬────────┘
        │               │                 │
        └───────────────┴─────────────────┘
                        │
                        ▼
                ┌───────────────┐
                │  inverter / meter   structs (estado global)
                └───────┬───────┘
                        │
       ┌────────────────┼─────────────────┐
       ▼                ▼                 ▼
 ┌──────────┐    ┌─────────────┐   ┌──────────────┐
 │ PID lib  │    │ pwmControl  │   │ Relés autom. │
 │ (Setpoint│───▶│ + manual    │──▶│ por % o W    │
 │  potTarg)│    │  fallback   │   │              │
 └──────────┘    └─────┬───────┘   └──────────────┘
                       │
                       ▼
                 ┌─────────────┐
                 │ ledcWrite() │ → triac / dimmer (GPIO 25, PWM 10-bit)
                 │ + DAC ch.2  │ → opcional 0-3.3 V
                 └─────────────┘
```

## 4. Ciclo de vida del firmware

### 4.1 `setup()` (FreeDS.ino:820)

```
1. Watchdog hw_timer (30 s) → resetModule()
2. Init ADC1 (12 bits, atten 11 dB)
3. Pines de relés a OUTPUT/LOW
4. EEPROM.begin(sizeof(config))   ← config ≈ 2 KB
   └─ checkEEPROM()                ← migración v0x0A → v0x17
5. SPIFFS.begin()  → readLanguages()
6. ledcAttachPin / ledcSetup       ← canal PWM 2 a 30 kHz por defecto
7. dac_output_enable(DAC_CHANNEL_2)
8. Init OLED (SSD1306 0x3c, I2C 4/15) + logo
9. EEPROM no inicializada ⇒ defaultValues() + saveEEPROM()
10. WiFi:
    · si !config.flags.wifi  ⇒  modo AP "FreeDS" + Captive Portal (DNS 53)
    · si  config.flags.wifi  ⇒  WiFiMulti, intenta ssid1, ssid2
       └─ si conecta:
          · NTP (configTzTime), mDNS, fauxmo, mqtt setup
          · UART1 (Modbus RS485 RX1=19/TX1=23)
          · UART2 (ESP-01 Solax v2,  RX=17/TX=5)
          · esp32ModbusTCP si wversion ∈ MODBUS_TCP rango
          · DallasTemperature (OneWire en GPIO 2)
          · setWebConfig()  ← registra TODAS las rutas HTTP
          · alexaConfig() / alexaStart() ← UPnP descubrible por Alexa
          · myPID.SetMode(AUTOMATIC|MANUAL), Setpoint = potTarget
          · GoodWe ⇒ inverterUDP.begin(8899)
          · xTimerStart(startTimer, 45 s) ← gracia para estabilizar lecturas
11. configureTickers() ← registra los 7 callbacks (todos disabled)
```

### 4.2 `loop()` (FreeDS.ino:1061)

```c
loop():
  timerWrite(timer, 0);          // patear al watchdog
  updateUptime();
  if (Flags.firstInit) dnsServer.processNextRequest();   // captive portal
  else                 fauxmo.handle();                  // descubrimiento Alexa

  if (config.flags.wifi && !Flags.firstInit) {
      Tickers.update();          // dispara callbacks programados
      if (processData) processingData();   // parser HTTP async
      changeScreen();            // botón PRG en GPIO0
      if (wversion == GOODWE)  parseUDP();
      if (PID auto && !errores && PID.Compute())  writePwmValue(PIDOutput);
      if (sensorTemperatura)   checkTemperature();
      if (debug4)              printDebug();
      if (NTPok)               checkTimer(); updateLocalTime();
      if (Flags.setBrightness) saveEEPROM(); display.setBrightness();
      if (oledAutoOff timeout) turnOffOled();
      if (potManPwmActive)     pwmManAuto switch when wsolar < potManPwm;
  }
```

El loop **no** bloquea en lecturas: las fuentes lentas (HTTP, Modbus,
sensores) corren en callbacks del scheduler con periodos configurados.

### 4.3 Scheduler `TickerScheduler`

Configurado en `configureTickers()` (FreeDS.ino:808). 7 slots fijos:

| Slot | Periodo            | Callback           | Propósito                                 |
|------|--------------------|--------------------|-------------------------------------------|
| 0    | 400 ms             | `showOledData()`   | Refresco pantalla OLED                    |
| 1    | 500 ms             | `every500ms()`     | SSE web + pinza + `pwmControl()` + PID input |
| 2    | 5 000 ms           | `connectToMqtt()`  | Reconexión MQTT (auto-disable al conectar) |
| 3    | 5 000 ms           | `connectToWifi()`  | Reconexión WiFi (auto-disable al conectar) |
| 4    | `getDataTime` (250-1500 ms) | `getSensorData()` | Lee la fuente activa (`config.wversion`)  |
| 5    | `publishMqtt` (1.5-60 s) | `publishMqtt()`   | Publica todos los tópicos                 |
| 6    | 1 000 ms           | `every1000ms()`    | Cálculo kWh y temperaturas                |

Estados típicos:
- Sin WiFi: solo el slot 0 (display) y el 3 (reintento WiFi).
- WiFi OK + MQTT OFF: 0, 1, 4, 6.
- WiFi OK + MQTT ON: 0, 1, 4, 5, 6.

### 4.4 Watchdog

`setup()` arma `hw_timer_t *timer` con prescaler 240 (1 µs) y alarma a
30 000 000 µs ⇒ **30 s de margen**. Cada iteración de `loop()` llama
`timerWrite(timer, 0)` para patear al perro. Si el loop se atasca o tarda
más de 30 s, se llama a `resetModule()` (ISR `IRAM_ATTR`) que invoca
`ESP.restart()`.

## 5. Modelo de estado y concurrencia

Toda la lógica vive en el contexto del *core 1* (loop principal de Arduino).
La concurrencia se limita a:

- **ISR del watchdog** (`resetModule`): solo reinicia.
- **Timers FreeRTOS** (`xTimerCreate`):
  - `relayOnTimer` / `relayOffTimer` (5 s, one-shot): re-arman las flags
    `Flags.RelayTurnOn` / `Flags.RelayTurnOff` para no encadenar conmutaciones
    de relés.
  - `startTimer` (45 s, one-shot): marca `Flags.bootCompleted = true` después
    del periodo de gracia para estabilización de la pinza amperimétrica.
- **Callbacks asíncronos** del `AsyncWebServer` y `AsyncMqttClient` y
  `AsyncTCP` (que sí corren en otra tarea); **escriben** en variables
  globales sin protección explícita pero, al ser el ESP32 *single core* en
  la app y los datos ser principalmente flags/estructuras simples, en la
  práctica funciona.

> ⚠️ Si añades funcionalidad pesada en callbacks asíncronos, considera
> serializar el trabajo a `loop()` con flags como ya hace `processData`
> (asyncHttpClient.ino).

## 6. Estado global (variables compartidas)

Todas declaradas en `FreeDS.ino` (líneas ~120 - 600). Las relevantes:

| Variable        | Tipo / tamaño  | Descripción |
|-----------------|---------------|-------------|
| `config`        | `struct CONFIG` (≈2 KB, persistido) | Toda la configuración del usuario y contadores kWh |
| `inverter`      | `struct INVERTER` | Lectura más reciente del inversor (W, V, A, SoC, batería, etc.) |
| `meter`         | `struct METER`    | Lectura del contador externo (RTU/TCP/MQTT) |
| `temperature`   | `struct TEMPERATURE_CONFIG` | Buffer de IDs OneWire + 3 lecturas |
| `pwm`           | `struct PWM_CONFIG` | `targetPwm`, `invert_pwm` (0-1023), `pwmValue` (0-100) |
| `Flags`         | `union 32-bit`     | Bits volátiles del runtime (firstInit, Updating, RelayXAuto/Man, ntpTime, …) |
| `Error`         | `union 16-bit`     | Bits de error (WiFi, MQTT, Recepción/Variación de datos, sensores) |
| `webMonitorFields` | `union 32-bit`  | Qué campos enviar al cliente web (depende de `wversion`) |
| `slave`         | `struct SLAVE`     | Estado cuando actúa como esclavo de otro FreeDS |
| `lang`          | `struct lang`      | Cadenas i18n cargadas de SPIFFS |
| `myPID`         | `PID`              | Controlador PID (Kp, Ki, Kd configurables) |
| `mqttClient`    | `AsyncMqttClient`  | Cliente MQTT |
| `server`        | `AsyncWebServer(80)` | Servidor HTTP |
| `events` / `webLogs` | `AsyncEventSource` | Streams SSE para datos en vivo y log |
| `modbustcp`     | `esp32ModbusTCP*`  | Cliente Modbus TCP (lazy, según `wversion`) |
| `Tickers`       | `TickerScheduler(7)` | Scheduler cooperativo |

## 7. Modos de funcionamiento

Definidos en `include/workingmode.h`. La selección es a través de
`config.wversion` y se agrupa por categorías de **20 IDs**:

```
RANGOS                     ID         Implementación
─────────                  ────       ──────────────
MODBUS_RTU      [ 1..20]    1-4       UART1 (RX1=19, TX1=23) → modbus.ino + modbus_functions.ino
HTTP_API        [21..40]   21-28      Cliente AsyncTCP (asyncHttpClient.ino) o UART2/UDP especiales
MQTT_MODE       [41..60]   41-42      Suscripciones a topics (mqtt.ino)
MODBUS_TCP      [61..80]   61-80      esp32ModbusTCP (modbustcp.ino)
```

Esto permite testar el rango con `if (wversion >= MODBUS_RTU && wversion <=
MODBUS_RTU + MODE_STEP - 1)`. Más detalles en
[INVERTERS.md](INVERTERS.md).

## 8. Lazo de control PWM (resumen)

Combina **PID** + reglas heurísticas en `pwmControl()` (pwm.ino:23) y
`every500ms()`:

1. **Validación de datos**: si `inverter.wgrid` no varía durante
   `maxErrorTime` ms ⇒ `Error.VariacionDatos = true`. Si no llegan datos
   ⇒ `Error.RecepcionDatos = true`. Cualquiera apaga el PWM por seguridad.
2. **Modo manual** (`pwmMan` o `pwmManAuto`): sube/baja `targetPwm` en
   pasos de 8 (de 0 a 1023, o de 209 a `maxPwmLowCost` en dimmers low cost),
   limitándose por `maxWattsTariff`.
3. **Modo automático**: el PID corre en `loop()` cuando hay condiciones
   válidas:
   - Setpoint = `config.potTarget` (W de excedente objetivo).
   - Input = `inverter.wgrid` (on-grid) o `inverter.batteryWatts`
     (off-grid).
   - Output (0..1023 o 209..maxPwmLowCost) → `ledcWrite(2, value)` y
     `dac_output_voltage(DAC_CHANNEL_2, value/4)`.
4. **Salidas auxiliares (4 relés)**: encendido/apagado por umbrales en
   `pwm.pwmValue` (% PWM) o por `inverter.wgrid` (W). Cada conmutación se
   throttea con `relayOnTimer` / `relayOffTimer` (5 s).
5. **Protección por temperatura**: si DS18B20 del termo supera
   `temperaturaApagado`, `Flags.tempShutdown = true` y `shutdownPwm()`.

Diagrama de estados (modo PWM):

```
            ┌────────────────────┐ pwmEnabled=false   ┌───────────┐
            │     OFF            │ ◀───────────────── │   AUTO    │
            │ ledcWrite(2,0)     │                    │  PID lib  │
            │ relays MAN only    │ pwmEnabled=true,   │  active   │
            └─────────▲──────────┘ !manual            └─────┬─────┘
                      │                                     │
   pwmEnabled=false   │                pwmMan=true /        │ pwmMan=true
       error          │                temp> apagado        ▼
                      │                                ┌───────────┐
                      └────────────────────────────────│  MANUAL   │
                                                       │ ramp ±8   │
                                                       └───────────┘
```

## 9. Manejo de errores y recuperación

Patrón general: **bit flag + timer**. Cada error mantiene un timestamp
(`timers.ErrorXxx`) y se considera activo cuando `millis() - timestamp >
config.maxErrorTime` (10–60 s). Las acciones típicas:

| Error                   | Detección                     | Reacción                                    |
|-------------------------|-------------------------------|---------------------------------------------|
| WiFi caída              | Evento `STA_DISCONNECTED`     | `shutdownPwm(true)`, deshabilita tickers, intenta reconexión cada 5 s |
| MQTT caída              | `onMqttDisconnect`            | Reintenta cada 5 s; sigue funcionando local |
| Sin datos del inversor  | Sin nuevas tramas en `maxErrorTime` | Apaga PWM, marca pantalla |
| Datos congelados        | `wgrid` inalterado en `maxErrorTime` | Apaga PWM |
| Temperatura sensor      | `getTempC` devuelve -127      | Apaga PWM si supera margen |
| Update OTA              | Multipart `/update`           | Desactiva todos los tickers excepto OLED, reset al final |
| Watchdog                | Loop bloqueado >30 s          | `ESP.restart()` |

## 10. Aspectos de seguridad

- **Autenticación HTTP**: `checkAuth()` (Basic Auth) en cada handler. Usuario fijo `admin`, password en `config.password` (almacenado en base64; no es cifrado real).
- **No HTTPS**: el ESP32 ofrece HTTP plano, asume red local.
- **OTA sin firma**: cualquier `.bin` válido se acepta. Restringido tras `checkAuth`.
- **Captive portal sin auth** (es lo esperado: el usuario aún no ha configurado).

## 11. Convenciones del código

- **Sin namespaces ni clases propias**: todo son funciones globales C-style.
- **Logging**: macro `INFOV(...)` (Support_functions.ino:639) — wrapper
  sobre `vsprintf` que envía la salida a `Serial` y/o al stream SSE
  `webLogs` según `config.flags.serial` y `config.flags.weblog`. Nunca uses
  `Serial.printf` para mensajes de usuario; usa `INFOV`.
- **Strings i18n**: `lang._XXX_` (cargadas en `readLanguages()`). Solo unas
  pocas cadenas están internacionalizadas (las de la pantalla OLED y mensajes
  básicos); la web tiene su propio JSON por idioma.
- **Comentarios**: bilingües ES/EN, principalmente ES en el `.ino`.

Para detalles módulo a módulo, ver [MODULES.md](MODULES.md).
