# Configuración persistente y migraciones de EEPROM

La configuración del dispositivo se almacena en una **EEPROM emulada**
(NVS / flash) como un volcado binario de la `struct CONFIG` declarada
en `FreeDS.ino:289-441`. La estructura tiene un padding final
(`uint8_t free[1042]`) para acomodar nuevos campos sin reformatear.

## 1. Estructura `CONFIG` (resumen por bloques)

```c
struct CONFIG {
  byte eeinit;                    // versión del esquema (0x17 actual)
  uint8_t wversion;               // ID del modo de fuente

  // Red
  char ip[16], gw[16], mask[16], dns1[16], dns2[16];

  // Salidas / relés
  int16_t potTarget;              // setpoint del PID (W de excedente)
  uint16_t R{01..04}Min;          // umbral % PWM para encender el relé
  int16_t  R{01..04}PotOn/Off;    // umbrales en W (si Min == 999)
  RelayFlags relaysFlags;         // bits "modo manual persistente"

  // Origen Solax v2
  char ssid_esp01[30], password_esp01[30];

  // Origen genérico
  char sensor_ip[30];

  // MQTT
  char MQTT_broker[25], MQTT_user[20], MQTT_password[20];
  uint16_t MQTT_port;
  char R01_mqtt[50], R02_mqtt[50], R03_mqtt[50], R04_mqtt[50];
  char password[30];              // Web UI (base64)
  char Solax_mqtt[50], Meter_mqtt[50];
  unsigned long publishMqtt;

  // WiFi STA
  char ssid1[30], pass1[30], ssid2[30], pass2[30], hostServer[12];

  // Pantalla
  uint8_t oledBrightness;

  // Temporizadores
  unsigned long oledControlTime, freeTemp, maxErrorTime, getDataTime;

  // PWM
  uint8_t  manualControlPWM, autoControlPWM;
  uint16_t pwmFrequency;
  uint16_t potManPwm;             // umbral W para potManPwmActive

  // Meter externo
  uint16_t baudiosMeter;
  uint8_t  idMeter;

  // Programador horario
  uint16_t timerStart, timerStop; // formato HHMM

  // Sensor temperatura
  uint8_t  temperaturaEncendido, temperaturaApagado, modoTemperatura;
  uint8_t  termoSensorAddress[8], triacSensorAddress[8], customSensorAddress[8];
  char     nombreSensor[30];

  // Slave PWM mínimo
  uint8_t  pwmSlaveOn;

  // Flags del sistema (32 bits, ver §3)
  SysBitfield flags;

  // Domoticz
  uint16_t domoticzIdx[3];
  uint16_t attachedLoadWatts;
  uint16_t maxPwmLowCost;

  // Energy meter
  float KwToday, KwExportToday, KwYesterday, KwExportYesterday, KwTotal, KwExportTotal;

  // Off-grid
  uint8_t soc;
  int16_t battWatts;

  // Time / i18n
  char    tzConfig[30];
  char    language[5];

  // Tarifa eléctrica
  uint16_t maxWattsTariff;

  // ICC Solar
  char  SoC_mqtt[30];
  float batteryVoltage;

  // NTP
  char  ntpServer[30];

  // Pinza
  float clampCalibration, clampVoltage;

  // PID
  float PIDValues[3];

  // Off-grid voltage offset
  float voltageOffset;

  // Misceláneos
  uint8_t gridPhase;
  uint8_t solaxVersion;

  // Padding para futuras versiones sin migrar
  uint8_t free[1042];
};
```

## 2. Versionado (`config.eeinit`)

Cada modificación incompatible incrementa `eepromVersion`:

```c
#define eepromVersion 0x17        // FreeDS.ino:23
```

`checkEEPROM()` (Support_functions.ino:846) aplica las migraciones desde
`0x0A` hasta `0x17` paso a paso. Cuando añadas un nuevo campo:

1. Sube `eepromVersion` en `FreeDS.ino` (p. ej. `0x18`).
2. Añade un caso a `checkEEPROM()`:
   ```c
   if (config.eeinit == 0x17) {
       config.miNuevoCampo = valorPorDefecto;
       config.eeinit = 0x18;
   }
   ```
3. Añade el reset a `defaultValues()` para los dispositivos nuevos.
4. Añade el procesador / handler web si lo expones por UI.
5. Considera añadir la publicación MQTT en `topicRegisters[]`.

## 3. Bitfields (`SysBitfield flags`, `Flags`, `Error`)

### `config.flags` (persistido)

```
bit  0  wifi              Hay credenciales válidas (true tras handleNetConfig)
bit  1  dhcp              IP dinámica (false ⇒ usa ip/gw/mask)
bit  2  mqtt              Cliente MQTT activo
bit  3  pwmEnabled        Control PWM global ON
bit  4  pwmMan            PWM en modo manual
bit  5  oledPower         Pantalla encendida
bit  6  oledAutoOff       Auto-apagado tras `oledControlTime`
bit  7  potManPwmActive   Activar manual cuando wsolar < potManPwm
bit  8  serial            Imprimir INFOV en Serial
bit  9  debug1            Verbosidad debug (varios bits)
bit 10  weblog            Replicar INFOV en SSE /weblog
bit 11  timerEnabled      Programador horario
bit 12  debug2
bit 13  sensorTemperatura DS18B20 activo
bit 14  alexaControl      fauxmoESP enabled
bit 15  domoticz          Integración con Domoticz
bit 16  dimmerLowCost     Modo dimmer 209..maxPwmLowCost
bit 17  changeGridSign    Inversión del signo de wgrid
bit 18  debug3
bit 19  debug4
bit 20  flipScreen        Voltea OLED
bit 21  offGrid           Modo aislado
bit 22  showEnergyMeter   Mostrar contador kWh en web
bit 23  offgridVoltage    Trigger por voltaje (true) o SoC (false)
bit 24  debug5
bit 25  useClamp          Usa pinza para currentCalcWatts
bit 26  useSolarAsMPTT    Victron: sumar acIn/acOut a wsolar
bit 27  useBMV            Victron: sumar wsolar a battery
bit 28  useExternalMeter  Hay un meter externo además del inversor
bit 29-30  spare
bit 31  debugPID
```

### `Flags` (volátil) — `FreeDS.ino:159-187`

`firstInit, Updating, flash, reboot, RelayTurnOn/Off, Relay{01..04}{Auto,Man}, setBrightness, weblogConnected, ntpTime, timerSet, pwmIsWorking, pwmManAuto, showClampCurrent, bootCompleted, tempShutdown`.

### `Error` — `FreeDS.ino:190-203`

`ConexionWifi, RecepcionDatos, ConexionMqtt, VariacionDatos, RemoteApi, temperaturaTermo, temperaturaTriac, temperaturaCustom`.

## 4. Backup / restore

- `GET /downloadBackup` ⇒ guarda en SPIFFS un volcado binario de
  `config` y lo descarga.
- `POST /backup` ⇒ escribe el binario subido a `/config.bin`, llama a
  `readConfigSpiffs()` que lo carga sobre `config`, hace `saveEEPROM`
  y reinicia.

## 5. Reset a fábrica

- Web: `/factoryDefaults` (POST) ⇒ `defaultValues() + restartFunction`.
- Botón PRG (GPIO 0) mantenido > 10 s en boot ⇒ idem (Support_functions.ino:209).
