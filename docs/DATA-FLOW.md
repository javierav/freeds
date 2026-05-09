# Flujo de datos

Cómo viaja un valor desde el inversor/contador hasta la salida (triac /
relés) y la interfaz web. Este documento es un paseo cronológico por la
información que circula por FreeDS.

## 1. Visión global

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FUENTES DE DATOS                             │
│  ESP01 UART · HTTP API · Modbus RTU · Modbus TCP · MQTT · UDP       │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ getSensorData()  (slot 4 ticker)
                                 ▼
                ┌────────────────────────────────────┐
                │ struct INVERTER + struct METER     │   ← variables globales
                │ (wgrid, wsolar, batterySoC, …)     │
                └────────┬───────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
   ┌───────────┐  ┌─────────────┐  ┌───────────────┐
   │   PWM     │  │   RELÉS     │  │  WEB / MQTT   │
   │ pwmControl│  │  por % o W  │  │  publishers   │
   └─────┬─────┘  └──────┬──────┘  └───────┬───────┘
         │                │                 │
         ▼                ▼                 ▼
    Triac/Dimmer      Cargas               Cliente final
   (GPIO 25 LEDC)   (GPIO 13/12/14/27)    (browser, broker, Alexa, Domoticz)
```

## 2. Adquisición: la pipe `getSensorData()`

```
┌─ ticker slot 4 (every config.getDataTime ms) ─┐
│  getSensorData()  (Support_functions.ino:21)  │
└───────────────────┬───────────────────────────┘
                    ▼
       switch (config.wversion)
           │
   ┌───────┼───────────────────────────────────────────────────────┐
   │       │                                                       │
   │ SOLAX_V2                  → readESP01()  (UART2 sync)         │
   │ SOLAX_V2_LOCAL/V1/WIBEEE  → runAsyncClient() (AsyncTCP)       │
   │ SHELLY_EM/FRONIUS_API     │  └─ payload acumulado en buffer   │
   │ SLAVE_MODE                │     y processData=true            │
   │                           │                                   │
   │ DDS238/DDSU666/SDM/MUST   → readModbus() → modbusSend()       │
   │ SMA/VICTRON/HUAWEI/...    │     UART1 (RX1=19,TX1=23)         │
   │ FRONIUS_MODBUS/SOLAREDGE  │                                   │
   │                           │                                   │
   │ GOODWE                    → sendUDPRequest()  UDP:8899        │
   │                                                               │
   │ MQTT_BROKER / ICC_SOLAR   → no se invoca aquí; los datos      │
   │                             llegan vía onMqttMessage()        │
   └───────────────────────────────────────────────────────────────┘
```

**Síncrono vs asíncrono**:
- **Síncrono** (en el callback del slot 4): UART (Solax v2 y Modbus
  RTU). Ejecutan dentro del scheduler y ya están bien acotados.
- **Asíncrono**: HTTP (AsyncTCP) y MQTT. Disparan callbacks en su tarea
  de stack lwIP. El parser pesado se difiere a `loop()` mediante el flag
  `processData`.

## 3. Estado de los datos

Los datos válidos viven en dos `struct` globales declarados en `FreeDS.ino`:

```c
struct INVERTER {
  float pv1c, pv2c, pv1v, pv2v;   // strings FV
  float pw1, pw2;                  // potencia por string
  float gridv;                     // tensión de red
  float wsolar, wtoday;            // potencia y energía solar
  float wgrid, wtogrid;            // intercambio con red (signo configurable)
  float wgrid_control;             // copia para detectar variación
  float temperature;               // temperatura inversor
  float batteryWatts, batterySoC;
  float loadWatts;
  float currentCalcWatts;          // calculado (pinza o senoide)
  float acIn, acOut;               // Victron
} inverter;

struct METER {
  float energyTotal, voltage, current;
  float activePower, aparentPower, reactivePower;
  float powerFactor, frequency;
  float importActive, exportActive;
  float importReactive, exportReactive;
  float phaseAngle;
  uint8_t read_state;              // contador de registros Modbus
  uint8_t send_retry;              // reintentos
} meter;
```

**Convención de signos** (clave): `inverter.wgrid > 0` significa, por
defecto, **vertido a red** y `< 0` consumo. Si tu inversor invierte la
convención, marca **Cambiar signo de red** (`config.flags.changeGridSign`).
Casi todos los parsers aplican `if (changeGridSign) wgrid *= -1` para
homogeneizar.

## 4. Validación en `pwmControl()`

En cada iteración del slot 1 (500 ms) se evalúan dos guardas:

| Condición                                       | Flag activado          | Acción                |
|-------------------------------------------------|------------------------|------------------------|
| `inverter.wgrid` no cambia en `maxErrorTime` ms | `Error.VariacionDatos` | `shutdownPwm(true)`   |
| Sin nuevas lecturas en `maxErrorTime` ms        | `Error.RecepcionDatos` | `shutdownPwm(true)`   |

`maxErrorTime` (10–60 s) es el principal "fusible": si la fuente de
datos calla, el firmware **deja de inyectar potencia** al triac.

## 5. Lazo PID

```
                  config.potTarget (W de excedente deseado, p.ej. 60 W)
                     ┌──────────────┐
   inverter.wgrid ──▶│  Setpoint    │
   (o batteryWatts)  │     -        │
                     │  Input       │      Kp Ki Kd  → tunable por consola
                     │  + ─────────▶│ PID  ─────────▶ Output (0..1023)
                     └──────────────┘                  │
                                                       ▼
                                              ledcWrite(2, val)   GPIO 25 (PWM)
                                              dac_output(2, val/4) GPIO 26 (DAC 8-bit)
```

- Si `changeGridSign` está activo, el PID corre en modo `DIRECT`,
  si no, `REVERSE`.
- Si modo manual (`pwmMan`), el PID se pone en `MANUAL` (Output = 0,
  Setpoint = 0) y el target se calcula con rampa lineal en `pwmControl()`.
- En **off-grid**: el input pasa a ser `inverter.batteryWatts` y el
  setpoint, el watt mínimo deseado para no descargar batería; las
  condiciones de pasar de manual a auto cambian (ver `pwm.ino:91-113`).

Los parámetros `PIDValues[3]` están persistidos y se ajustan vía la
consola web con `tunePID 0.05;0.06;0.03`.

## 6. Salidas auxiliares (4 relés)

Lógica en `pwm.ino`:

```
Para cada relé i (1..4):
  Si en MAN (Flags.RelayXMan o config.relaysFlags.RXMan = true):
      → siempre ON (relayManualControl)
  Si NO MAN:
      Si config.RXMin != 999:           // umbral por % PWM
          ON  cuando pwmValue >= RXMin
          OFF cuando pwmValue <= (RXMin - 10)
      Si config.RXMin == 999:           // umbral por W
          ON  cuando wgrid > RXPotOn (o < si changeGridSign)
          OFF cuando wgrid < RXPotOff
  Anti-rebote: enableRelay/disableRelay con xTimer (5 s)
```

Cada cambio publica su nuevo estado vía MQTT (`config.RXX_mqtt`).

## 7. Difusión de datos al cliente

### 7.1 Server-Sent Events

`AsyncEventSource events("/events")`. Cada 500 ms (slot 1)
`every500ms()` llama `sendEvents()`:

```
events.send(printUptime(), "uptime");        // string corto
events.send(sendJsonWeb(), "jsonweb");       // JSON con todo el estado
```

`sendJsonWeb()` consulta `webMonitorFields.X` para incluir solo los
campos relevantes a la fuente actual y construye el JSON con
`ArduinoJson` (1024 B). Esto evita gastar ancho de banda en campos
que el inversor no aporta.

### 7.2 MQTT (`publishMqtt()`)

`publishMqtt()` (mqtt.ino:556) publica:

| Topic (raíz `<host>`)                    | Contenido                                     |
|-------------------------------------------|-----------------------------------------------|
| `/<topicRegisters[i].topics>`             | Cada registro de la tabla declarativa         |
| `/pwm`                                    | `pwmValue` 0..100                              |
| `/stat/pwm`                               | `AUTO` / `MAN` / `OFF`                         |
| `/relay/{1..4}/STATUS` (o el topic configurado) | `ON` / `OFF`                            |
| `/Meter` (modo Modbus RTU)                | JSON con `Power, Voltage, Current, ...`        |
| `domoticz/in`                             | Si Domoticz activo, IDX configurados           |

### 7.3 Weblog SSE

`webLogs` (`/weblog`) reproduce todo lo emitido por `INFOV()`. Solo
arranca cuando el cliente se conecta (`Flags.weblogConnected`).

## 8. Persistencia y reset

```
RAM (config struct) ─────save────▶ EEPROM emulada (sectores flash)
                       ◀───load──── (al boot, EEPROM.get(0, config))
                                     │
                                     └─ checkEEPROM() migra versiones
                                        (eepromVersion = 0x17)
```

`saveEEPROM()` se llama tras cada cambio relevante en handlers HTTP,
MQTT, Alexa o consola. El campo `config.eeinit` actúa de barrera —
si la versión almacenada no coincide con `eepromVersion`,
`checkEEPROM` aplica las migraciones acumuladas paso a paso.

Los contadores `KwToday/KwExportToday/KwTotal/...` se persisten
también, pero se actualizan **constantemente** mientras hay datos
NTP. A las 00:00 se mueve `KwToday → KwYesterday` y se hace
`saveEEPROM`. En cortes de luz, los kWh del día en curso pueden
perderse parcialmente (no hay flush en cada incremento).

## 9. Diagrama de máquinas de estado

### 9.1 Modo de trabajo del PWM

```
   pwmEnabled=false                  pwmEnabled=true && pwmMan=true
   ──────────────────────────────────────────────────────────────
   ┌────────┐                       ┌────────────┐
   │  OFF   │ ◀─────────────────── │  MANUAL    │
   └────┬───┘                       └─────▲──────┘
        │ pwmEnabled=true,                │ pwmMan=true
        │ !pwmMan                         │
        ▼                                 │
   ┌────────────┐  pwmMan=false /  ───────┘
   │  AUTO PID  │  pwmMan=true
   │  (PID lib) │
   └────────────┘
```

Triggers que fuerzan **OFF**:
- `Error.RecepcionDatos`, `Error.VariacionDatos`, `!Flags.pwmIsWorking`.
- `Flags.tempShutdown` (temp. termo > apagado).
- WiFi disconnected (eventos lwIP).
- OTA en curso (`Flags.Updating`).
- Comando MQTT/web `pwm 0` o `factoryDefaults`.

### 9.2 Conectividad

```
                       wifi=false
                       (factory)
                          │
                          ▼
                 ┌────────────────┐
                 │   AP "FreeDS"  │
                 │ Captive Portal │
                 │  (DNS + HTTP)  │
                 └───────┬────────┘
                  Save credentials
                          │
                          ▼
                 ┌────────────────┐
                 │  STA WiFi      │     5s reintento si caída
                 │  WiFiMulti     │ ◀───────────────────────
                 └───────┬────────┘
                  STA_GOT_IP
                          │
                          ▼
                 ┌────────────────┐         ┌──────────────┐
                 │ Servicios online│ ───────▶│ MQTT connect │
                 │ Tickers.enable  │ disable │  (5 s retry)  │
                 │ NTP, mDNS, web  │ ◀───────└──────────────┘
                 └────────────────┘
```

## 10. Mapa de threads

| "Thread"                       | Origen                              | Quién es propietario       |
|--------------------------------|-------------------------------------|----------------------------|
| `loop()` (core 1)              | Arduino main                        | App principal              |
| AsyncTCP / WebServer / SSE     | Tarea de `AsyncTCP` (lwIP core)     | Llamada por callbacks      |
| AsyncMqttClient                | Tarea propia de MQTT                | Callbacks `onMqtt*`        |
| FreeRTOS Timers (relayOn/Off)  | Servicio de timers                  | `enableRelay/disableRelay` |
| ISR `resetModule`              | hw_timer                            | Watchdog                   |
| FreeRTOS WiFi event task       | esp32                               | `WiFiEvent()`              |

> Todos estos contextos terminan **escribiendo en variables globales sin
> mutex**. Funciona porque las modificaciones suelen ser asignaciones
> atómicas (uint32, float, bools) y porque el consumidor (`loop()`)
> revalida con timestamps. Mantén el patrón si extiendes el código.

## 11. Resumen — referencias rápidas

- Punto de entrada: `setup()` en `FreeDS.ino:820`.
- Bucle principal: `loop()` en `FreeDS.ino:1061`.
- Selector de fuente: `getSensorData()` en `Support_functions.ino:21`.
- Validación de datos: comienzo de `pwmControl()` en `pwm.ino:23`.
- Generador JSON web: `sendJsonWeb()` en `webserver_handlers.ino:317`.
- Publicador MQTT: `publishMqtt()` en `mqtt.ino:556`.
