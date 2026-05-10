# Mapa de módulos (`src/*.ino`)

Resumen de cada fichero del firmware con sus funciones públicas más
relevantes y cómo se usan. Las líneas son referencias a la versión
`1.0.7 rev2`.

> Recordatorio: todos los `.ino` se compilan en una sola unidad de
> traducción, por lo que **cualquier función definida aquí es visible para
> el resto**.

## `FreeDS.ino` (1138 líneas) — núcleo del firmware

Contiene:
- Las **estructuras globales**: `config` (CONFIG, ~2 KB persistido), `inverter`,
  `meter`, `pwm`, `Flags`, `Error`, `webMonitorFields`, `lang`, `temperature`,
  `slave`, `uptime`, `logMessage`.
- La **definición de pines** (PWM, RX/TX, relés, ADC) y los `#define` clave
  (`OLED`, `MAX_SCREENS`, `eepromVersion`).
- La **clase `CaptiveRequestHandler`** que sirve la página inicial
  HTML inline para configurar la red en modo AP.
- `defaultValues()`: valores iniciales de `config`.
- `configureTickers()`: registra los 7 callbacks del scheduler.
- `setup()` (l. 820) y `loop()` (l. 1061).
- ISR `resetModule()` del watchdog.
- Construcción de `myPID` (PID_v1) y de `inverterUDP` para GoodWe.

## `Support_functions.ino` (930 líneas) — utilidades, logging, energy monitor

Funciones que el resto de módulos usan:

| Función                      | Qué hace                                                                                  |
|------------------------------|-------------------------------------------------------------------------------------------|
| `getSensorData()`            | **Dispatcher** según `config.wversion` (Solax v2/v1, Wibeee, Shelly, Fronius, Modbus, GoodWe). Llamado en slot 4 del scheduler. |
| `setGetDataTime()`           | Ajusta el periodo del slot 4 a un mínimo seguro por modo                                  |
| `every500ms()` / `every1000ms()` | Lógicas periódicas (ver ARCHITECTURE.md §4.3)                                          |
| `restartFunction()`          | Reinicio ordenado tras `saveEEPROM()`                                                     |
| `saveEEPROM()`               | `EEPROM.put + commit`                                                                     |
| `checkEEPROM()`              | **Migración** entre versiones (0x0A → 0x17). Si añades campos a `config`, sube `eepromVersion` y añade un caso aquí. |
| `defaultValues()`            | Reseteo a fábrica (también llamado al cambiar de mayor a `eeinit` desconocido)            |
| `updateUptime()`, `printUptime()`, `printUptimeOled()` | Cálculo y formateo del uptime (con manejo de rollover de `millis()`)        |
| `updateLocalTime()`, `checkTimer()` | NTP y programador horario (timerStart/timerStop)                                   |
| `INFOV(...)`                 | **Logger** principal (Serial + SSE webLogs)                                               |
| `addLog`, `sendWeblogStreamTest` | Buffer circular de 20 mensajes para el log web                                       |
| `calcWattsToday()`           | Integral discreta de `wgrid` para acumular `KwToday/KwExportToday/KwTotal`. Reset a las 00:00 |
| `defineWebMonitorFields(version)` | Bitfield de qué campos enviar en JSON al frontend según el origen de datos          |
| `current()`, `calcIrms()`    | Cálculo RMS de la pinza SCT-013 (basado en EmonLib). 1484 muestras ⇒ ~131 ms              |
| `readClamp()`                | Llama a `calcIrms` y calcula `inverter.currentCalcWatts = Irms * clampVoltage`             |
| `readLanguages()`            | Carga `lang-XX.json` desde SPIFFS al struct `lang`                                         |
| `writeConfigSpiffs/readConfigSpiffs` | Backup/restore de la EEPROM en SPIFFS (`/config.bin`)                              |
| `bootTimer()`                | Callback del `startTimer` de 45 s ⇒ `Flags.bootCompleted = true`                           |

## `inverter.ino` (203 líneas) — parsers de inversores no-Modbus

Trabaja con `DynamicJsonDocument root(4096)` global (definido en `FreeDS.ino`).

| Función           | Origen / formato                                                          |
|-------------------|---------------------------------------------------------------------------|
| `readESP01()`     | Lee línea JSON desde `SerieEsp` (UART2) – Solax V2 con ESP01 puenteando WiFi del inversor |
| `parseJson()`     | Solax V2 (15 campos por índice)                                           |
| `parseJsonv1()`   | Solax V1 (HTTP `/api/realTimeData.htm`)                                   |
| `parseJsonv2local()` / `parseJsonv3local()` | Solax local (POST `optType=ReadRealTimeData`)            |
| `parseJsonFronius()` | Fronius solar API `/solar_api/v1/GetPowerFlowRealtimeData.fcgi`         |

Todos terminan con `Error.RecepcionDatos = false; timers.ErrorRecepcionDatos = millis()`.

## `asyncHttpClient.ino` (353 líneas) — cliente HTTP no bloqueante

Patrón `AsyncTCP`:
1. `runAsyncClient()`: arma una petición (URL específica por `wversion`),
   gestiona timeouts (`CONNECTION_TIMEOUT=30s`, `RECEIVING_DATA_TIMEOUT=10s`)
   y delega a callbacks `onConnect/onData/onDisconnect/onError/onTimeout`.
2. **Reensambla los chunks**: `message.message[]` (5000 bytes), busca
   `Content-Length:` y `\r\n\r\n` para localizar el payload, soporta
   transferencias en varios `onData()`.
3. Cuando se completa, marca `processData = true` y `loop()` llama a
   `processingData()` que despacha al parser correspondiente.
4. Incluye una implementación KMP de `strstr` (búsqueda en buffers binarios).

> Esta es la pieza menos obvia: separar **recibir** (callbacks asíncronos)
> de **parsear** (síncrono en loop) evita reentrancia con ArduinoJson y
> mantiene el WDT contento.

## `modbus.ino` (381 líneas) — Modbus RTU sobre RS485

Maneja **DDS238-2 ZN/S, DDSU666, SDM120/220, MustSolar** y **delega** al
resto de plantas Modbus TCP (que también pasan por `readModbus()` aunque su
implementación esté en `modbustcp.ino`).

Para cada meter define la tabla de registros y la rutina específica
(`dds2382`, `ddsu666`, `sdm120`, `mustSolar`). Ciclo:

```
loop modbus():
   if data_ready
       parse buffer
       meter.read_state++  (puntero al siguiente registro de la tabla)
       set Error.RecepcionDatos = false
   else if !send_retry  (5 reintentos)
       modbusSend(idMeter, FC, addr, length)
```

`modbusSend` y `modbusReceiveBuffer` viven en `modbus_functions.ino` (frame
RTU + CRC16).

## `modbus_functions.ino` (149 líneas) — capa de transporte Modbus RTU

Implementa la mecánica del protocolo Modbus RTU sobre `SerieMeter` (UART1):
- `modbusSend(slaveAddr, fc, regAddr, count)` — construye trama, calcula
  CRC16 y la escribe en la UART.
- `modbusReceiveReady()` — true cuando la trama de respuesta es completa y
  el CRC cuadra.
- `modbusReceiveBuffer(buf, regCount)` — copia la respuesta cruda al buffer.

Si añades un meter Modbus, este fichero **no** suele tocarse.

## `modbustcp.ino` (762 líneas) — Modbus TCP

Estado: usa `esp32ModbusTCP` (lazy `new` en `setup()` y al cambiar de
versión). Define una tabla `registerData[]` por inversor:

```c
struct registerData {
    float *variable;       // dónde escribir
    uint8_t serverID;
    uint16_t address;
    uint16_t length;
    valueType type;        // U16FIX0, S32FIX3, F32FIX0, … o tipos especiales (FRONIUSPV1, SUNNYBOYGRID, …)
};
```

Inversores cubiertos: **SMA Sunny Boy / Sunny Island, Victron, Fronius
Modbus, Huawei, SolarEdge, Wibeee Modbus, Schneider, Ingeteam**.

`configModbusTcp()` registra el callback de respuesta y arma el bucle de
peticiones. El callback re-inyecta los valores en `inverter`/`meter`
según el `valueType`.

## `mqtt.ino` (646 líneas) — WiFi events + MQTT cliente y broker mode

Tres responsabilidades:

1. **WiFi**: `connectToWifi()`, `errorConnectToWifi()`, `WiFiEvent()`
   (callback de eventos del stack lwIP).
2. **MQTT cliente**:
   - `connectToMqtt()`, `onMqttConnect/Disconnect/Message`.
   - Suscripciones por defecto: `<host>/cmnd/{pwm,pwmman,pwmmanvalue,
     screen,pwmfrec,brightness,pwmvalue}` y `<host>/relay/{1..4}/CMND`.
   - **Modo `MQTT_BROKER`**: se suscribe a topics configurables
     (`Solax_mqtt`, `Meter_mqtt`) y parsea JSON tipo Tasmota.
   - **Modo `ICC_SOLAR`**: se suscribe a `Inverter/...` con topics
     planos (un valor por topic, ASCII).
   - **Domoticz**: si `flags.domoticz`, suscribe a `domoticz/out` y
     publica en `domoticz/in` con los IDX configurados.
3. **Publisher**: `publishMqtt()` itera sobre `topicRegisters[]` (tabla
   declarativa: variable + nombre de topic) y publica todo.

Tabla `topicRegisters` añadible — basta añadir una línea para publicar un
nuevo float por MQTT.

## `pwm.ino` (425 líneas) — control de salida (triac + relés)

- `pwmControl()` (l. 23): el lazo de control. Implementa los modos
  manual y automático y dispara los relés (4 salidas auxiliares) según %
  PWM o W de red, con histéresis y delay anti-rebote (timers FreeRTOS).
- `shutdownPwm(forceRelayOff, message)`: apaga PWM y opcionalmente todos
  los relés. Llamada desde múltiples sitios cuando hay errores.
- `writePwmValue(value)`: escribe simultáneamente al canal LEDC 2 y al
  DAC2 (GPIO 26).
- `calculeTargetPwm(percent)`: convierte 0-100 % a 0-1023 (modo normal) o
  209-`maxPwmLowCost` (dimmer chino con escalón).
- `calcPwmProgressBar()`: calcula `pwm.pwmValue` (0-100) para mostrar.
- `relayManualControl(forceOFF)`: estado de los 4 relés según
  `Flags.RelayXMan` y `config.relaysFlags.RXMan`.
- `enableRelay()` / `disableRelay()`: callbacks de los timers FreeRTOS
  para anti-rebote.

## `display.ino` (332 líneas) — pantalla OLED

`showOledData()` se llama cada 400 ms (slot 0) y dibuja una de
**MAX_SCREENS** pantallas (botón PRG las cicla):

| screen | Contenido                                              |
|--------|--------------------------------------------------------|
| 0      | Principal (Solar / Red, PWM, modo)                     |
| 1      | Datos meter (modos Modbus RTU): V, I, kWh, factor      |
| 2      | Datos string (PV1/PV2)                                 |
| 3      | Estado relés y temperatura inversor                    |
| 4      | Energy meter (kWh hoy / ayer / total)                  |
| 5      | Temperaturas DS18B20 (termo, triac, custom)            |

También incluye `showLogo()`, `turnOffOled()`, símbolos i18n, y bitmaps
desde `include/bitmap.h`.

## `webserver_handlers.ino` (1429 líneas) — todas las rutas HTTP

Es el módulo más grande. Estructura:

1. **Handlers de POST** específicos:
   - `handleNetConfig` — `/handleNetConfig` (config WiFi, IP estática,
     hostname).
   - `handleMqttConfig` — `/handleMqttConfig` (broker, credenciales,
     topics relés, Domoticz IDX).
   - `handleConfig` — `/handleConfig` (off-grid, sensor temp, alexa,
     password, baudios meter, idmeter, etc.).
   - `handleControlConfig` — `/handleControlConfig` (PWM target,
     umbrales relés, programador, frecuencia PWM, low-cost).
2. **Generadores JSON**:
   - `sendJsonWeb()` ⇒ payload del SSE `events` (estado, errores,
     campos visibles según `webMonitorFields`).
   - `sendMasterData()` ⇒ payload del endpoint `/masterdata`
     consumido por instancias en modo `SLAVE_MODE`.
3. **`setWebConfig()`** (l. 640): registra TODAS las rutas en `server`:
   - HTML procesado por plantilla (`index.html`, `Red.html`,
     `Mqtt.html`, `Config.html`, `Salidas.html`, `Ota.html`,
     `weblog.html`) — el procesador inyecta valores y vive en
     `webserver_processors.ino`.
   - Recursos estáticos comprimidos (`*.jgz` con cabecera
     `Content-Encoding: gzip`).
   - Endpoints de control: `/reboot`, `/selectversion`, `/language`,
     `/brightness`, `/tooglebuttons`, `/handlecmnd` (consola
     interactiva), `/factoryDefaults`.
   - OTA: `/update` (multipart, soporta `firmware.bin` y
     `spiffs.bin`), `/backup`, `/downloadBackup`.
   - SSE: `events` (datos en vivo) y `webLogs` (consola en vivo).
4. **Alexa (fauxmoESP)**: `alexaConfig()` registra los hooks
   pre/post-handler para que UPnP/SSDP inyecte sus respuestas.
   `alexaStart()` añade 3 dispositivos virtuales (PWM ON/OFF,
   Manual+%, Oled).

> El comando consola `/handlecmnd` es la **superficie de control
> avanzada**. Cada `if (comando == "...")` es un comando expuesto. Buen
> sitio para añadir flags experimentales.

## `webserver_processors.ino` (806 líneas) — *template processors*

Cada handler `request->send(SPIFFS, ".html", "text/html", false,
processorXxx)` invoca uno de estos procesadores. Reciben un placeholder
`%TOKEN%` y devuelven la cadena que lo sustituye. Un ejemplo: en
`processorFreeDS()`, el token `%FREEDSVERSION%` se reemplaza por
`String(version) + " " + String(beta)`.

Los procesadores conocen los campos del struct `config` y los renderizan
con `selected="..."` en `<select>`, `checked` en `<input type=checkbox>`,
etc. Si añades opciones nuevas a una página, hay que actualizar el
procesador correspondiente.

## `goodwe.ino` (113 líneas) — protocolo UDP propietario GoodWe

- `sendUDPRequest()`: envía `AA 55 C0 7F 01 06 00 02 45` al puerto 8899
  del inversor.
- `parseUDP()`: decodifica la respuesta byte a byte (offsets 7..87)
  para extraer PV1/PV2, batería, red, frecuencia, kWh diario y
  temperatura.

## `shelly.ino` (53 líneas) — Shelly EM (HTTP /emeter/N)

`parseShellyEM()` se llama dos veces alternativamente para los dos
canales del Shelly EM (uno para red, otro para FV).

## `wibeee.ino` (54 líneas) — Wibeee (XML /en/status.xml)

`parseWibeee()` extrae con `midString()` los campos `<faseN_*>`. Usa
Fase 1 como medida de red y Fase 2 como medida del inversor.

## `master_freeds.ino` (111 líneas) — modo Slave

Cuando `wversion == SLAVE_MODE`, este FreeDS pide GET `/masterdata` a
otro FreeDS (`config.sensor_ip`) y replica sus datos
(`slave.masterMode`, `slave.masterPwmValue`). Si el master tiene PWM
por debajo de `pwmSlaveOn`, este esclavo apaga su salida.

## `tempsensor.ino` (110 líneas) — DS18B20 OneWire

- `buildSensorArray()`: enumera los IDs OneWire detectados en GPIO 2 y
  los publica para el dropdown de la web.
- `calcDallasTemperature()`: lee las 3 direcciones configurables
  (`termoSensorAddress`, `triacSensorAddress`, `customSensorAddress`).
- `checkTemperature()`: aplica la regla de apagado del PWM según
  `modoTemperatura` (1=auto, 2=manual, 3=ambos), umbrales
  `temperaturaEncendido/Apagado`.

---

Para añadir un nuevo módulo de inversor, ver receta paso a paso en
[EXTENDING.md](EXTENDING.md).
