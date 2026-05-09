# Recetas para extender FreeDS

Cómo añadir funcionalidades comunes sin reescribir medio firmware. Cada
sección lista los archivos a tocar y el orden recomendado.

## 1. Añadir un nuevo inversor / contador

### 1.1 Asignar un ID

Edita `include/workingmode.h` y elige un ID **dentro del rango** que
mejor describa el transporte:

```c
// HTTP_API 21..40
#define MI_INVERTER 29
```

> ⚠️ Mantén el rango: `getSensorData()`, `setGetDataTime()`,
> `defineWebMonitorFields()` y muchos handlers usan rangos para detectar
> categorías. Añadir un Modbus TCP fuera de 61..80 romperá la lógica.

### 1.2 Implementar el parser

Crea `src/miinverter.ino` (Arduino lo concatenará automáticamente) con:

```c
void parseMiInverter(char *data) {
    DeserializationError error = deserializeJson(root, data);
    if (error) { INFOV("parseMiInverter() failed: %s\n", error.c_str()); return; }
    inverter.wgrid    = (float) root["grid"];
    inverter.wsolar   = (float) root["solar"];
    inverter.wtoday   = (float) root["energy_today"];
    if (config.flags.changeGridSign) inverter.wgrid *= -1.0;
    Error.RecepcionDatos = false;
    timers.ErrorRecepcionDatos = millis();
}
```

### 1.3 Conectar al pipeline

- Si HTTP: añade un `case MI_INVERTER:` en `runAsyncClient()`
  (asyncHttpClient.ino:108-138) con la URL, y otro en
  `processingData()` (asyncHttpClient.ino:281).
- Si Modbus RTU: añade un `case` en `readModbus()` (modbus.ino:332)
  apuntando a una nueva función `miInverter()` con su tabla de
  registros.
- Si Modbus TCP: define una tabla `registerData miRegisters[]` en
  `modbustcp.ino` y añade `case MI_INVERTER:` en el dispatcher.
- Si MQTT: añade un `case` en `onMqttMessage()` y en
  `suscribeMqttMeter()` / `unSuscribeMqtt()`.
- Si UDP: imita la estructura de `goodwe.ino`.

### 1.4 Activar campos visibles

Añade un caso en `defineWebMonitorFields()` (Support_functions.ino:450)
con el bitfield apropiado (ver tabla de bits en
[INVERTERS.md](INVERTERS.md)).

### 1.5 Frecuencia mínima

`setGetDataTime()` (Support_functions.ino:60) — añade tu mínimo seguro.

### 1.6 Exposición en la UI

- `webserver_processors.ino` — añade tu opción al `<select>` de origen
  en `processorConfig` o donde corresponda.
- Añade un caso en `display.ino` si quieres una pantalla específica.

### 1.7 Persistencia

Si necesitas un campo en `config` (p.ej. credenciales del inversor),
sigue [CONFIG-EEPROM.md §2](CONFIG-EEPROM.md#2-versionado-configeeinit)
(sube `eepromVersion` y añade caso en `checkEEPROM`).

## 2. Añadir una nueva pantalla en el OLED

1. Sube `MAX_SCREENS` en `FreeDS.ino:69`.
2. Añade un `case <n>:` en `showOledData()` (`display.ino`).
3. Si la pantalla solo tiene sentido en ciertos modos, ajusta los
   filtros en `mqtt.ino:onMqttMessage` (cmnd `screen`) y en
   `Support_functions.ino:changeScreen` para saltarla.

## 3. Añadir un comando MQTT/Web

### Tópico MQTT

1. En `onMqttConnect()` (`mqtt.ino:213`) añade el `topics[]`:
   ```c
   static char topics[][12] = {"pwm","pwmman", …, "miCmd"};
   ```
2. En `onMqttMessage()` añade:
   ```c
   sprintf(tmpTopic, "%s/cmnd/miCmd", config.hostServer);
   if (strcmp(topic, tmpTopic) == 0) {
       config.miCampo = atoi(payload);
       saveEEPROM();
       return;
   }
   ```

### Comando consola web

Añade un bloque en `webserver_handlers.ino` dentro de `/handlecmnd`:

```c
if (comando == "miCmd") {
    if (payload != "miCmd") {
        config.miCampo = payload.toFloat();
        saveEEPROM();
        INFOV("miCmd set to %f\n", config.miCampo);
    } else {
        INFOV("miCmd: %f\n", config.miCampo);
    }
}
```

Convención: si solo se manda `miCmd` sin argumento, el handler
**imprime** el valor actual (modo lectura). Útil para debug.

## 4. Publicar una nueva variable por MQTT

Edita la tabla `topicRegisters` en `mqtt.ino:27`:

```c
topicData topicRegisters[] = {
    …,
    &miNuevaVariable, "miNombre"
};
```

Se publicará automáticamente en `<host>/miNombre` con 2 decimales.

## 5. Añadir campos al SSE web (`/events`)

`sendJsonWeb()` (`webserver_handlers.ino:317`) genera el JSON. Inserta
un `jsonValues["mi_campo"] = ...` y consume el campo en
`data/freeds.min.js` (frontend) — recuerda regenerar/actualizar SPIFFS.

## 6. Añadir un nuevo idioma

1. Crea `languages/lang-XX.json` y `lang-XX.js` siguiendo el mismo
   esquema que los existentes (25 cadenas indexadas).
2. Empaqueta los `.jgz` correspondientes y añádelos a SPIFFS (carpeta
   `data/`) — el script de PlatformIO `pio run -t buildfs` lo hace.
3. En `webserver_handlers.ino`, registra la ruta:
   ```c
   server.on("/lang-XX.js", HTTP_GET, [](AsyncWebServerRequest *request) { … });
   ```
4. En `processorConfig`, añade la opción `<option value="XX">…</option>`.

## 7. Cambiar la placa hardware

Edita los `#define` de pines en `FreeDS.ino:67-108` (bloque `OLED`/no
OLED). Los más relevantes:

- `pin_pwm` (default 25)
- `pin_rx`, `pin_tx` (UART2, ESP-01)
- `RX1`, `TX1` (UART1, RS485)
- `PIN_RL1..PIN_RL4`
- `ADC_INPUT` (pinza)
- `DS18B20` (OneWire)

Si tu placa no tiene OLED, comenta el `#define OLED` (FreeDS.ino:67).
Verifica que las funciones `display.*` queden compiladas dentro de
`#ifdef OLED` en `display.ino` (algunas ya lo están).

## 8. Subir el binario / SPIFFS por primera vez

Con PlatformIO instalado:

```bash
unzip lib.zip                 # extrae las dependencias bajo lib/
pio run                       # compila firmware
pio run -t upload             # flashea por USB
pio run -t buildfs            # genera spiffs.bin
pio run -t uploadfs           # graba SPIFFS
```

Después, futuras actualizaciones se pueden hacer **OTA** desde la
web (`/Ota.html` → subir `firmware.bin` y `spiffs.bin`).

## 9. Buenas prácticas

- **Logs con `INFOV(...)` siempre**, no `Serial.printf` directamente
  (te quedas sin weblog).
- **Después de modificar `config.X`, llama a `saveEEPROM()`** o el
  cambio se perderá en el próximo reset.
- **No bloquees el `loop()`**: cualquier I/O potencialmente largo va
  como callback (TickerScheduler, AsyncTCP) o se difiere con flag.
- **Watchdog: 30 s**. Operaciones largas (ej. `Update.write` para OTA)
  suelen ya patear el watchdog implícitamente; otras (escaneo WiFi,
  parsers grandes) deben respetar el límite.
- **Sin RTOS mutex**: las variables globales se manipulan desde
  callbacks asíncronos. Usa asignaciones atómicas (uint32, bool, float)
  o flags + procesado en `loop()`.
- **No introduzcas nuevos `delay()` largos** en el `loop()` o en
  callbacks; rompen la fluidez del SSE.

## 10. Esqueleto de un módulo nuevo

```c
// src/miinverter.ino
/*
  miinverter.ino - FreeDs Mi Inverter Support
*/

void miInverterRequest(void) {
    // armar petición y enviarla
}

void miInverterParse(char *data) {
    DeserializationError err = deserializeJson(root, data);
    if (err) { INFOV("miInverter() %s\n", err.c_str()); return; }

    inverter.wgrid  = (float) root["grid"];
    inverter.wsolar = (float) root["solar"];
    if (config.flags.changeGridSign) inverter.wgrid *= -1.0;

    Error.RecepcionDatos = false;
    timers.ErrorRecepcionDatos = millis();
}
```

Engancha `miInverterRequest()` desde `getSensorData()` y
`miInverterParse()` desde donde llegue la respuesta.

---

¿Dudas? Empieza por leer [ARCHITECTURE.md](ARCHITECTURE.md) y
[DATA-FLOW.md](DATA-FLOW.md), seguro localizas lo que buscas.
