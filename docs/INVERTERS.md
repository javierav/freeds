# Fuentes de datos soportadas

FreeDS abstrae múltiples inversores y contadores tras un único campo
(`config.wversion`) cuyos rangos están definidos en
`include/workingmode.h`:

```c
#define MODE_WIDTH 80
#define MODE_STEP  20
#define MODBUS_RTU 1   // 1..20
#define HTTP_API   21  // 21..40
#define MQTT_MODE  41  // 41..60
#define MODBUS_TCP 61  // 61..80
```

## Catálogo

| ID  | Nombre constante     | Categoría    | Transporte             | Implementación                               |
|-----|----------------------|--------------|------------------------|----------------------------------------------|
| 1   | `DDS238_METER`       | Modbus RTU   | UART1 RS485 (FC 0x03)  | `modbus.ino: dds2382()`                      |
| 2   | `DDSU666_METER`      | Modbus RTU   | UART1 RS485 (FC 0x04)  | `modbus.ino: ddsu666()`                      |
| 3   | `SDM_METER`          | Modbus RTU   | UART1 RS485 (FC 0x04)  | `modbus.ino: sdm120()`                       |
| 4   | `MUSTSOLAR`          | Modbus RTU   | UART1 RS485            | `modbus.ino: mustSolar()`                    |
| 21  | `SOLAX_V2`           | HTTP API     | UART2 ⇄ ESP-01 puente  | `inverter.ino: readESP01() + parseJson()`    |
| 22  | `SOLAX_V2_LOCAL`     | HTTP API     | HTTP POST en LAN        | `inverter.ino: parseJsonv2local/v3local()`   |
| 23  | `SOLAX_V1`           | HTTP API     | HTTP GET                | `inverter.ino: parseJsonv1()`                |
| 24  | `WIBEEE`             | HTTP API     | HTTP GET XML            | `wibeee.ino: parseWibeee()`                  |
| 25  | `SHELLY_EM`          | HTTP API     | HTTP GET JSON           | `shelly.ino: parseShellyEM()` (alterna 2 sensores) |
| 26  | `FRONIUS_API`        | HTTP API     | HTTP GET JSON           | `inverter.ino: parseJsonFronius()`           |
| 27  | `SLAVE_MODE`         | HTTP API     | HTTP GET `/masterdata`  | `master_freeds.ino: parseMasterFreeDs()`     |
| 28  | `GOODWE`             | HTTP API     | UDP propietario :8899   | `goodwe.ino: sendUDPRequest/parseUDP`        |
| 41  | `MQTT_BROKER`        | MQTT         | Tópicos JSON tipo Tasmota | `mqtt.ino: onMqttMessage()`                |
| 42  | `ICC_SOLAR`          | MQTT         | Tópicos planos          | `mqtt.ino: onMqttMessage()`                  |
| 61  | `SMA_BOY`            | Modbus TCP   | TCP :502                | `modbustcp.ino: smaBoy()`                    |
| 62  | `VICTRON`            | Modbus TCP   | TCP :502                | `modbustcp.ino: victron()`                   |
| 63  | `FRONIUS_MODBUS`     | Modbus TCP   | TCP :502                | `modbustcp.ino: fronius()`                   |
| 64  | `HUAWEI_MODBUS`      | Modbus TCP   | TCP :502                | `modbustcp.ino: huawei()`                    |
| 65  | `SMA_ISLAND`         | Modbus TCP   | TCP :502                | `modbustcp.ino: smaIsland()`                 |
| 66  | `SCHNEIDER`          | Modbus TCP   | TCP :502                | `modbustcp.ino: schneiderModbus()`           |
| 67  | `WIBEEE_MODBUS`      | Modbus TCP   | TCP :502                | `modbustcp.ino: wibeeeModbus()`              |
| 68  | `INGETEAM`           | Modbus TCP   | TCP :502                | `modbustcp.ino: ingeteamModbus()`            |
| 80  | `SOLAREDGE`          | Modbus TCP   | TCP **:1502**           | `modbustcp.ino: solarEdge()`                 |

> El bloque MODBUS_TCP detecta `wversion == SOLAREDGE` para usar el
> puerto especial 1502. Si añades un inversor que requiera otro puerto,
> imita ese caso especial en `setup()` (FreeDS.ino:970-972) y
> `/selectversion` (webserver_handlers.ino).

## Frecuencias mínimas

`setGetDataTime()` (Support_functions.ino:60) impone un mínimo seguro
para el slot 4 según la categoría. Si añades un nuevo modo, agrega aquí
su mínimo (250–1500 ms suelen ser razonables).

## Cobertura de campos

`defineWebMonitorFields(version)` (Support_functions.ino:450) decide qué
campos del JSON web están "activos" para cada modo. La codificación es
un **bitfield 32-bit** (struct `webMonitorFields` en FreeDS.ino:254).
Por ejemplo:

```c
case SHELLY_EM:        webMonitorFields.data = 0x00580322; break;
case FRONIUS_API:      webMonitorFields.data = 0x00700000; break;
case INGETEAM:         webMonitorFields.data = 0x0F77E006; break;
```

Los bits relevantes:

| Bit  | Macro             | Significado                     |
|------|-------------------|----------------------------------|
| 0    | `energyTotal`     | Energía total acumulada         |
| 1    | `voltage`         | V de meter                      |
| 2    | `current`         | A de meter                      |
| 3    | `activePower`     | W activa                        |
| 4    | `aparentPower`    | VA aparente                     |
| 5    | `reactivePower`   | VAR reactiva                    |
| 6    | `powerFactor`     | cos φ                            |
| 7    | `frequency`       | Hz                               |
| 8    | `importActive`    | kWh importados                  |
| 9    | `exportActive`    | kWh exportados                  |
| 13   | `pv1c` … 16: pv2v | Strings FV                       |
| 17   | `pw1`, 18: `pw2`  | Potencia por string             |
| 19   | `gridv`           | V red                            |
| 20   | `wsolar`          | Potencia FV                      |
| 21   | `wtoday`          | kWh diarios                      |
| 22   | `wgrid`           | Intercambio con red              |
| 23   | `wtogrid`         | Vertido diario                   |
| 24   | `temperature`     | T inversor                       |
| 25   | `batteryWatts`    | Batería W                        |
| 26   | `batterySoC`      | Batería SoC                      |
| 27   | `loadWatts`       | Consumo casa                     |

Si tu nueva fuente proporciona, p.ej., `wsolar + wgrid`, el bitfield
sería `(1<<20) | (1<<22) = 0x00500000`. El frontend sabrá no esperar
los demás campos.

## Cómo descubrir endpoints reales

| Modo            | Detalle                                                |
|------------------|--------------------------------------------------------|
| Solax v2 ESP-01  | El ESP-01 que viene con el inversor expone tópico WiFi (`SSID: SOLAXX`) que se proxa por UART al ESP32 |
| Solax local      | POST `http://<ip>/?optType=ReadRealTimeData&pwd=admin` |
| Solax v1         | GET `http://<ip>/api/realTimeData.htm`                 |
| Wibeee           | GET `http://<ip>/en/status.xml`                        |
| Shelly EM        | GET `http://<ip>/emeter/0` y `/emeter/1` alternados   |
| Fronius API      | GET `http://<ip>/solar_api/v1/GetPowerFlowRealtimeData.fcgi` |
| Slave (FreeDS)   | GET `http://<ip>/masterdata`                           |
| GoodWe           | UDP :8899 con cabecera `AA 55 C0 7F 01 06 00 02 45`    |
| ICC Solar (MQTT) | Tópicos `Inverter/GridWatts`, `Inverter/MPPT1_Watts`, … |
| MQTT Broker      | Tópicos configurables `Solax_mqtt`, `Meter_mqtt` (formato Tasmota `ENERGY` JSON) |
