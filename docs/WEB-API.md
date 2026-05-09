# Web API y servidor HTTP

FreeDS embebe un `AsyncWebServer` en el puerto 80 que sirve la interfaz
web y expone endpoints REST/POST para configuración y control.

## 1. Stack web

```
Browser
  │
  ▼
HTTP/1.1  ─── AsyncWebServer ─── SPIFFS (archivos estáticos *.jgz)
              │
              ├─ Plantillas con %TOKENS% → procesadores en
              │  webserver_processors.ino (sustitución por config.X)
              │
              ├─ AsyncEventSource("/events")  → JSON cada 500 ms
              ├─ AsyncEventSource("/weblog")  → líneas INFOV() en vivo
              │
              ├─ POST handlers → modifican struct config + saveEEPROM()
              │
              └─ Multipart /update → Update.write() para OTA
```

## 2. Páginas servidas (SPIFFS)

| Ruta                | Fichero SPIFFS    | Procesador               |
|---------------------|-------------------|---------------------------|
| `/`                 | `index.html`      | `processorFreeDS`         |
| `/Red.html`         | `Red.html`        | `processorRed`            |
| `/Mqtt.html`        | `Mqtt.html`       | `processorMqtt`           |
| `/Config.html`      | `Config.html`     | `processorConfig`         |
| `/Salidas.html`     | `Salidas.html`    | `processorSalidas`        |
| `/Ota.html`         | `Ota.html`        | `processorOta`            |
| `/weblog.html`      | `weblog.html`     | `processorOta`            |

Recursos estáticos (`/sb-admin-2.min.js`, `/freeds.min.js`, etc.) se
cargan desde SPIFFS con la cabecera `Content-Encoding: gzip` (ficheros
con extensión `.jgz`).

## 3. Server-Sent Events

```
GET /events     event: uptime  data: "Fecha: dd/mm/yyyy …"
                event: jsonweb data: { "wgrid":…, "wsolar":…, … }
GET /weblog     event: weblog  data: "HH:MM:SS - mensaje INFOV"
```

El `JS` del frontend abre un `EventSource` por cada uno y actualiza la
UI sin necesidad de polling.

## 4. Endpoints REST

### Configuración (POST de formulario)

| Ruta                    | Handler                   | Body (form-urlencoded)                       |
|-------------------------|---------------------------|----------------------------------------------|
| `/handleNetConfig`      | `handleNetConfig`         | `wifi1`, `wifip1`, `wifi2`, `wifip2`, `host`, `dhcp`, `ip`, `gw`, `mask`, `dns1`, `dns2` |
| `/handleMqttConfig`     | `handleMqttConfig`        | `mqttactive`, `broker`, `mqttuser`, `mqttpass`, `mqttport`, `mqttpublish`, `mqttr1..r4`, `solax`, `meter`, `soctopic`, `domoticzactive`, `idxpwm`, `idxman`, `idxoled` |
| `/handleConfig`         | `handleConfig`            | `offGrid`, `soc`, `battWatts`, `changeGridSign`, `baudiosmeter`, `idmeter`, `wifis`, `oldpass`, `newpass`, `maxerrortime`, `getdatatime`, `autoPowerOff`, `sensorTemp`, `tempOn`, `tempOff`, `termoaddrs`, `triacaddrs`, `customaddrs`, `customSensor`, `alexa` |
| `/handleControlConfig`  | `handleControlConfig`     | `pwmactive`, `pottarget`, `r0Xmin`, `r0XpotOn/Off`, `loadwatts`, `wattstariff`, `slavepwm`, `manpwm`, `autopwm`, `potpwmactive`, `potmanpwm`, `maxpwmlowcost`, `lowcostactive`, `timeractive`, `timerStart/Stop`, `R0X_man`, `frecpwm` |
| `/factoryDefaults`      | (lambda)                  | —                                            |

### Acciones / botones

| Ruta              | Método | Acción                                                |
|-------------------|--------|-------------------------------------------------------|
| `/reboot`         | GET    | `saveEEPROM` y reinicio en 4 s                        |
| `/selectversion`  | POST   | `data` = nuevo `wversion`                             |
| `/language`       | POST   | `value` = código (`es`, `en`, `pt`, `ca`, `ga`)       |
| `/brightness`     | POST   | `data` = 0–100 brillo OLED                            |
| `/tooglebuttons`  | POST   | `data` ∈ 1..7 (toggle relé manual / OLED / PWM / Manual) |
| `/handlecmnd`     | POST   | `webcmnd` = `<comando> [valor]` (consola; ver §5)     |
| `/masterdata`     | GET    | JSON consumido por slaves                             |

### OTA y backup

| Ruta              | Método | Body                                                  |
|-------------------|--------|-------------------------------------------------------|
| `/update`         | POST   | multipart con `firmware.bin` o `spiffs.bin`           |
| `/backup`         | POST   | multipart con `config.bin` para restaurar             |
| `/downloadBackup` | GET    | descarga el config actual (`config_<host>_<ver>.bin`) |

## 5. Consola interactiva (`/handlecmnd`)

`webserver_handlers.ino:957-1188` contiene un dispatcher de comandos.
Cada uno se invoca como `<nombre> [argumento]`. Los más importantes:

| Comando                | Argumento                    | Efecto                                  |
|------------------------|------------------------------|-----------------------------------------|
| `rebootcause`          | —                            | Imprime causa del último reset          |
| `getfreeheap`          | —                            | Heap libre                              |
| `serial 0/1`           | bool                         | Activa/desactiva debug por Serial       |
| `debug 0..6`           | int                          | Selecciona el bit de debug              |
| `weblog 0/1`           | bool                         | Activa el log SSE                       |
| `KwToday <Wh>`         | número                       | Inyecta valor manual al contador        |
| `KwTotal/KwExportToday/KwExportTotal` | …             | Idem                                     |
| `flipScreen`           | —                            | Voltea pantalla                         |
| `gridPhase 1..3`       | int                          | Selecciona fase a usar                  |
| `showEnergyMeter 0/1`  | bool                         |                                         |
| `useExternalMeter 0/1` | bool                         |                                         |
| `solaxVersion 2/3`     | int                          | Variante v2/v3 del JSON Solax           |
| `tzConfig <CET-1...>`  | string POSIX TZ              | Cambia zona horaria (ver `TimeZones.txt`) |
| `ntpServer <host>`     | string                       |                                         |
| `offgridVoltage 0/1`   | bool                         | SoC vs voltaje en off-grid              |
| `voltageOffset <f>`    | float                        |                                         |
| `useClamp 0/1`         | bool                         | Usa pinza para `currentCalcWatts`       |
| `clampCalibration`/`clampVoltage` | float             |                                         |
| `showClampCurrent 0/1` | bool                         | Imprime I medida en log                 |
| `maxWattsTariff <W>`   | int                          | Tope para modo manual                   |
| `tunePID 0.05;0.06;0.03` | string                     | Kp;Ki;Kd                                |
| `useSolarAsMPTT 0/1`   | bool                         | Victron                                 |
| `useBMV 0/1`           | bool                         | Victron                                 |
| `SetControllerDirection` | 0/1                        | DIRECT/REVERSE para PID                 |
| `pwmFrec <hz*10>`      | int                          | Frecuencia en décimas de Hz             |
| `listFiles` / `writeSpiffs` / `readSpiffs` | —      | Debug de SPIFFS                         |

> Cualquier comando nuevo que añadas debe respetar el patrón
> `if (comando == "miCmd") { ... }`. Si modifica `config`, llama a
> `saveEEPROM()`.

## 6. Autenticación

`checkAuth(request)` se invoca al inicio de cada handler relevante. Usa
HTTP Basic con usuario fijo `admin` y password en `config.password`
(almacenada como base64). El password por defecto es `YWRtaW4=` →
`admin`. `config.password` se actualiza en `/handleConfig` enviando
`oldpass` (debe coincidir) y `newpass`.

## 7. fauxmoESP (Alexa)

El handler `alexaConfig()` añade hooks `onRequestBody` y `onNotFound` al
servidor para responder a las consultas SSDP/UPnP de Alexa, sin
interferir con las rutas registradas. `alexaStart()` declara 3
"dispositivos virtuales":

```
"Derivador <MAC>"        → ON/OFF      (controla pwmEnabled)
"Derivador Manual <MAC>" → DIMMABLE    (controla pwmMan + manualControlPWM)
"Derivador Oled <MAC>"   → DIMMABLE    (controla oledPower + oledBrightness)
```

## 8. mDNS

`MDNS.begin(config.hostServer)` publica `<host>.local`. La cabecera
`Access-Control-Allow-Origin` se inicializa con esa URL para permitir
peticiones cross-origin desde `<host>.local`.

## 9. Captive portal

Activo solo cuando `config.flags.wifi == false`. Mounted via
`server.addHandler(new CaptiveRequestHandler).setFilter(ON_AP_FILTER)`.
Sirve un formulario inline (HTML embebido en `FreeDS.ino:631-651`) y
publica DNS catch-all en el puerto 53 (`dnsServer`).
