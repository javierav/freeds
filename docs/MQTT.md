# Integración MQTT

FreeDS puede actuar como **cliente MQTT** para publicar telemetría y
aceptar comandos, además de **subscribirse** a topics ajenos (Tasmota,
ICC Solar, Domoticz). Toda la lógica vive en `mqtt.ino`.

## 1. Conexión

- Cliente: `AsyncMqttClient mqttClient` (no bloqueante).
- ID: `config.hostServer` (p.ej. `freeds_a4c1`).
- Reintento cada 5 s desde `Tickers` slot 2 mientras esté en error.
- Credenciales y broker se editan en `/Mqtt.html`.
- KeepAlive 30 s.

## 2. Topics publicados (root = `<config.hostServer>`)

`publishMqtt()` itera sobre la tabla `topicRegisters` (mqtt.ino:27) y
publica cada flotante de `inverter`/`meter`/`temperature`/`config` con
2 decimales:

```
<host>/pw1, pw2, pv1v, pv1c, pv2v, pv2c, wsolar
<host>/invTemp, wtoday, wgrid, wtogrid, gridv
<host>/calcWatts, batteryWatts, batterySoC, loadWatts
<host>/AcIn, AcOut
<host>/voltage, current
<host>/tempTermo, tempTriac, tempCustom
<host>/KwToday, KwYesterday, KwExportToday, KwExportYesterday, KwTotal, KwExportTotal
```

Más:

```
<host>/pwm                   → 0..100 (porcentaje del triac)
<host>/stat/pwm              → "AUTO" | "MAN" | "OFF"
<host>/relay/{1..4}/STATUS   → "ON" | "OFF"     (topic configurable: config.RXX_mqtt)
<host>/Meter                 → JSON con campos del meter (solo modos Modbus RTU)
```

## 3. Topics suscritos para control

Al conectar, `onMqttConnect()` se suscribe a los siguientes (publicar
ahí permite controlar el dispositivo):

| Topic                          | Payload                | Acción                                       |
|--------------------------------|------------------------|----------------------------------------------|
| `<host>/cmnd/pwm`              | `0`/`1`                | Activa/desactiva el PWM globalmente          |
| `<host>/cmnd/pwmman`           | `0`/`1`                | Modo automático (`0`) o manual (`1`)         |
| `<host>/cmnd/pwmmanvalue`      | `0..100`               | Valor del PWM en modo manual                 |
| `<host>/cmnd/pwmfrec`          | `10..30000`            | Frecuencia (décimas de Hz)                   |
| `<host>/cmnd/screen`           | `0..MAX_SCREENS`       | Cambia pantalla OLED                         |
| `<host>/cmnd/brightness`       | `0..100`               | Brillo OLED                                  |
| `<host>/cmnd/pwmvalue`         | `0..1023`              | Valor crudo del PWM (debug)                  |
| `<host>/relay/{1..4}/CMND`     | `0`/`1`                | Encender/apagar relé manualmente             |

## 4. Modo `MQTT_BROKER` (wversion 41)

Cuando los datos del inversor/contador llegan **vía MQTT** (firmware
externo tipo Tasmota), se suscribe a:

```
config.Solax_mqtt   (default "solaxX1/tele/SENSOR")
config.Meter_mqtt   (default "meter/tele/SENSOR")
```

Espera JSON con la estructura Tasmota:

```json
{
  "ENERGY": {
    "Power": <W>,
    "Voltage": <V>,
    "Pv1Power": …, "Pv2Power": …,
    "Pv1Voltage": …, "Pv2Voltage": …,
    "Pv1Current": …, "Pv2Current": …,
    "Today": <kWh>,
    "Temperature": <°C>
  }
}
```

## 5. Modo `ICC_SOLAR` (wversion 42)

Topics planos (no JSON), un valor por mensaje:

```
Inverter/GridWatts
Inverter/MPPT1_Watts, MPPT2_Watts
Inverter/MPPT1_Volts, MPPT2_Volts
Inverter/MPPT1_Amps,  MPPT2_Amps
Inverter/PvWattsTotal
Inverter/SolarKwUse
Inverter/BatteryVolts, BatteryAmps, BatteryWatts
Inverter/LoadWatts
Inverter/Temperature
<config.SoC_mqtt>     (default "Inverter/BatterySOC")
```

## 6. Domoticz

Si `config.flags.domoticz`:

- Suscripción a `domoticz/out` (broadcast de cambios desde Domoticz).
- Reacciona al payload con `idx == config.domoticzIdx[N]`:
  - `idxpwm`  (slot 0): switch PWM ON/OFF.
  - `idxman`  (slot 1): selector + slider de manualControlPWM.
  - `idxoled` (slot 2): selector + slider de oledBrightness.
- En `publishMqtt()` re-publica el estado actual a `domoticz/in`.

## 7. Reglas de actualización

- `publishMqtt()` se invoca periódicamente desde el slot 5 cada
  `config.publishMqtt` ms (1.5 - 60 s).
- Cambios manuales de relé, PWM, etc. publican **inmediatamente** su
  nuevo estado (sin esperar al ticker), ver `pwm.ino` y `mqtt.ino`.
- Las funciones de publicación comprueban siempre:
  - `config.flags.mqtt`
  - `!Error.ConexionMqtt`
  - `strcmp("5.8.8.8", config.sensor_ip) != 0`  (placeholder de "no
    configurado", evita publicar basura).
