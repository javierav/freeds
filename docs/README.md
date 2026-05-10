# Documentación técnica de FreeDS

Esta carpeta contiene la documentación de arquitectura del firmware FreeDS
(versión `1.0.7 rev2`) — un derivador universal de excedentes solares basado
en ESP32 (board *Heltec WiFi Kit 32*).

> **Audiencia objetivo**: desarrolladores que aterrizan por primera vez en el
> proyecto y necesitan comprender cómo está organizado el código antes de
> añadir funcionalidades, integrar nuevos inversores, depurar problemas o
> portar el firmware a otro hardware.

## Índice

| Documento | Contenido |
|-----------|-----------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Visión global: capas, ciclo de vida, diagrama de bloques, scheduler cooperativo, watchdog, FSM principal |
| [MODULES.md](MODULES.md) | Descripción fichero a fichero (`src/*.ino`) y su responsabilidad |
| [DATA-FLOW.md](DATA-FLOW.md) | Flujo de datos entre fuente → PID → PWM/relés, estructuras compartidas, estados de error |
| [INVERTERS.md](INVERTERS.md) | Catálogo de fuentes de datos soportadas (Solax, Fronius, Victron, SMA, Huawei, SolarEdge, Shelly EM, Wibeee, ICC Solar, GoodWe, Schneider, Ingeteam, MustSolar, contadores DDS/SDM, modos MQTT/Slave) |
| [WEB-API.md](WEB-API.md) | Servidor HTTP, SSE (`/events`, `/weblog`), endpoints REST, OTA y Captive Portal |
| [MQTT.md](MQTT.md) | Tópicos publicados y suscritos, integración con Domoticz / ICC Solar / Tasmota, control por Alexa (fauxmoESP) |
| [HARDWARE.md](HARDWARE.md) | Mapa de pines, periféricos del ESP32, variantes de PCB, pinza amperimétrica, dimmer triac |
| [CONFIG-EEPROM.md](CONFIG-EEPROM.md) | Estructura `CONFIG`, versionado de la EEPROM, flags de bitfield, migraciones |
| [EXTENDING.md](EXTENDING.md) | Recetas para añadir funcionalidades (un nuevo inversor, una pantalla, un comando web, un tópico MQTT, un sensor) |
| [BUILD.md](BUILD.md) | Cómo compilar (PlatformIO), librerías incluidas en `lib.zip`, flashear firmware y SPIFFS |

## TL;DR de la arquitectura

```
┌─────────────────────────────────────────────────────────────────────┐
│                          ESP32 (Heltec WiFi Kit 32)                 │
│                                                                     │
│   ┌───────────────────┐   ┌────────────────────┐                    │
│   │   Fuentes datos   │──▶│  Lógica de control │──┐                 │
│   │  (inverter/meter) │   │    PID + reglas    │  │                 │
│   └───────────────────┘   └────────────────────┘  │                 │
│       ▲     ▲     ▲                ▲              ▼                 │
│       │     │     │                │      ┌──────────────┐          │
│  ESP01 RS485 TCP/UDP/HTTP/MQTT     │      │ Triac (PWM)  │──▶ Carga │
│                                    │      │ + 4 relés    │          │
│                                    │      └──────────────┘          │
│                                    │                                │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │ Servicios: WiFi · MQTT client · HTTP server (SPIFFS) · OTA │  │
│   │ mDNS · NTP · OneWire · OLED · fauxmoESP (Alexa) · Domoticz │  │
│   └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

- **Lenguaje**: C++ (Arduino framework) sobre ESP-IDF.
- **Build system**: PlatformIO (`platformio.ini`), placa `heltec_wifi_kit_32`.
- **Concurrencia**: cooperativa con `TickerScheduler` (7 slots) + tareas FreeRTOS para los timers de relés. Todo el control corre en `loop()`.
- **Persistencia**: `EEPROM` emulada (estructura `CONFIG`, ~2 KB) + `SPIFFS` para web estática y ficheros de idiomas.
- **UI**: pantalla OLED 128×64 (SSD1306) + interfaz web responsive (Bootstrap / SB-Admin-2) servida desde SPIFFS, comprimida en gzip (`*.jgz`), con SSE para actualización en vivo.

Lee [ARCHITECTURE.md](ARCHITECTURE.md) para el detalle completo.
