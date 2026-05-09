# Compilar y flashear FreeDS

## 1. Requisitos

- **VSCode** + extensión **PlatformIO IDE**.
- (alternativa CLI) `pip install platformio` y usar `pio` directamente.
- Driver USB-Serial CP210x o CH340 según la placa Heltec.

## 2. Preparación

```bash
git clone https://github.com/javierav/freeds.git
cd freeds
unzip lib.zip          # extrae las librerías a lib/
```

`lib.zip` contiene la versión exacta y *parcheada* de:

- `TickerScheduler` (con parche)
- `AsyncMqttClient`
- `AsyncTCP`
- `ESPAsyncWebServer`
- `esp32-oled-ssd1306`
- `esp32ModbusTCP`
- `OneWire` / `DallasTemperature`
- `PID_v1`
- `fauxmoESP`

Esto es importante: usar la versión "upstream" del registry suele
introducir incompatibilidades, especialmente con `TickerScheduler`.

## 3. `platformio.ini`

```ini
[env:heltec_wifi_kit_32]
platform   = espressif32 @ ~3.5.0
board      = heltec_wifi_kit_32
framework  = arduino
monitor_speed = 115200
extra_scripts = pio/name-firmware.py     ; renombra el bin tras compilar
board_build.f_flash  = 80000000L         ; 80 MHz flash
board_build.f_cpu    = 240000000L        ; 240 MHz CPU
board_build.flash_mode = dio
build_flags = -DCORE_DEBUG_LEVEL=5
              -DPIO_FRAMEWORK_ESP_IDF_ENABLE_EXCEPTIONS
              -DCONFIG_FREERTOS_ASSERT_DISABLE
              -DCONFIG_LWIP_ESP_GRATUITOUS_ARP
              -DCONFIG_LWIP_GARP_TMR_INTERVAL=30
lib_deps    = https://github.com/bblanchon/ArduinoJson.git
```

> ArduinoJson se descarga del registro; las demás se cargan
> automáticamente desde `lib/`.

## 4. Compilar

```bash
pio run                   # compila firmware → .pio/build/heltec_wifi_kit_32/firmware.bin
pio run -t buildfs        # genera spiffs.bin desde data/
```

`pio/name-firmware.py` (configurado vía `extra_scripts`) renombra el
binario a algo como `freeds-1.0.7-rev2.bin` para distinguir versiones.

## 5. Flashear por USB

Conecta la placa, identifica el puerto y:

```bash
pio run -t upload         # firmware
pio run -t uploadfs       # SPIFFS (necesario tras cambios en data/)
```

## 6. Flashear por OTA

Desde la web:

1. Navega a `http://<host>.local/Ota.html`.
2. Sube primero `firmware.bin`, luego `spiffs.bin`.
3. Pulsa `CTRL+F5` para forzar recarga del frontend.

> Orden importante: si sobreescribes SPIFFS antes que el firmware y la
> versión nueva añade campos al `config`, puedes encontrarte con la
> EEPROM en estado intermedio. Sigue siempre `firmware → spiffs`.

## 7. Estructura de SPIFFS (`data/`)

```
data/
├── index.html, Red.html, Mqtt.html, Config.html, Salidas.html,
│   Ota.html, weblog.html      ← plantillas con %TOKENS%
├── *.jgz                       ← JS, CSS, fuentes pre-comprimidas en gzip
├── lang-{ca,en,es,ga,pt}.{json,js.jgz}
├── webfonts/                   ← .woff/.woff2 sin gzip (binarios)
└── favicon.ico, freeds.png.jgz
```

> `.jgz` es solo `*.gz` renombrado para evitar la auto-descompresión de
> SPIFFS. El servidor añade `Content-Encoding: gzip` y el navegador lo
> descomprime al vuelo. Reduce drásticamente el espacio de SPIFFS.

## 8. Logs y depuración

- **USB Serial** a 115200 (`Serial Monitor` en VSCode).
- **Web**: `http://<host>.local/weblog.html` (SSE en vivo).
- Verbosidad: `debug 1..6` por la consola (cada nivel activa
  `config.flags.debugN`).

## 9. CI

`.travis.yml` define una CI básica con PlatformIO en Python 2.7 (legado
del momento de creación). Si reactivas CI, considera migrar a GitHub
Actions.

## 10. Estructura post-compilado

```
.pio/build/heltec_wifi_kit_32/
├── firmware.bin              ← lo que sube /update
├── firmware.elf
├── partitions.bin
├── bootloader.bin
└── spiffs.bin                ← solo tras `buildfs`
```

Si necesitas un único `.bin` "todo en uno" para flashear con `esptool`:

```bash
esptool.py --chip esp32 merge_bin -o freeds-merged.bin \
   --flash_mode dio --flash_size 4MB \
   0x1000 bootloader.bin \
   0x8000 partitions.bin \
   0x10000 firmware.bin \
   0x290000 spiffs.bin
```

(las direcciones dependen del particionado por defecto del board
`heltec_wifi_kit_32`; consúltalas con `pio run -v`.)
