# Hardware

FreeDS está diseñado para correr sobre la placa **Heltec WiFi Kit 32**
(ESP32 + OLED 128×64 SSD1306 integrado por I2C). El firmware asume esa
distribución de pines pero se puede portar a otras placas ajustando los
`#define` del bloque `OLED` en `FreeDS.ino:67-108`.

## 1. Mapa de GPIO

```
  GPIO   Función                       Notas
  ────   ────────────────────────────  ─────────────────────────────
   0     Botón PRG                     Pull-up interno; baja al pulsar
   2     OneWire DS18B20               #define DS18B20 2 (FreeDS.ino:71)
   4     SDA OLED I2C                  Heltec wiring
   5     UART2 TX                      → ESP-01 RX (Solax V2)
  12     Relé 2                        digitalWrite OUTPUT
  13     Relé 1
  14     Relé 3
  15     SCL OLED I2C
  16     OLED reset                    Pulsado al iniciar
  17     UART2 RX                      ← ESP-01 TX (Solax V2)
  19     UART1 RX1                     ← Convertidor RS485 (Modbus)
  23     UART1 TX1                     → Convertidor RS485 (Modbus)
  25     PWM (LEDC ch.2)               → triac / dimmer (carga principal)
  26     DAC2 (8-bit)                  → opcional 0-3.3V analógico
  27     Relé 4
  34     ADC1_CH6 (input only)         Pinza SCT-013-030 (12-bit)
```

`build_flags` en `platformio.ini` fija `CORE_DEBUG_LEVEL=5` y CPU a
240 MHz. La frecuencia del PWM por defecto es 30 kHz (configurable
10–30 000 con `pwmFrequency / 10.0`).

## 2. Periféricos del ESP32 usados

| Subsistema | Pin/Canal | Uso                              |
|------------|-----------|----------------------------------|
| ADC1 CH6   | 34        | Pinza amperimétrica (RMS)        |
| LEDC ch.2  | 25        | PWM 10-bit (0..1023)             |
| DAC ch.2   | 26        | Salida analógica (constrain 0-255) |
| UART0      | (Serial)  | Debug 115200 baud                |
| UART1      | 19/23     | RS485 Modbus RTU                 |
| UART2      | 17/5      | ESP-01 (Solax V2 puente)         |
| I2C        | 4/15      | OLED SSD1306                     |
| OneWire    | 2         | Hasta 15 sensores DS18B20        |
| Wi-Fi      | —         | Modo STA + AP                    |

## 3. Variantes de PCB

### 3.1 `clinxer/` (PCB original)
Diseñada para encajar en una caja **Sonoff** de ~40 mm de altura.
Incluye placa de potencia y de control apilables.

- Placa de potencia: triac + módulo MOC3041 o módulo dimmer chino
  (low-cost). Soldadura sin máscara para reforzar las pistas.
- Placa de control: ESP32-WROOM-32U (antena externa), módulo RS485,
  conectores para sensores DS18B20, relés externos, SCT-013, ventilador
  por relé 4 o 5 V.

### 3.2 `sanchez-rev1/`
Variante moderna en KiCad con dos PCBs y novedades:

- Conectores RS485, sensores T, 4 relés.
- Conectores para pinza de jack 3.5 mm o de 2 pines.
- Ventilador controlado **por PWM** (cuanto mayor el PWM, más caudal).
- Gerbers compilados para fabricación directa.

### 3.3 `KRIDA Electronics/`
Documentación / firmware del módulo dimmer Krida. Incluye dos firmwares
para el ATmega8/328P del módulo (`OLD_PWM16A_*`, `NEW_PWM16A_*`).

### 3.4 `Clinxer - Version ACS712/`
Variante con sensor ACS712 (efecto Hall) en lugar de SCT-013 para medir
corriente. Incluye instrucciones (PDF).

## 4. Pinza amperimétrica SCT-013-030

Conectada al ADC1_CH6 (GPIO 34). Cálculo en `Support_functions.ino`:

```c
calcIrms(unsigned int Number_of_Samples = 1484)   // ~131 ms
  // ADC1 12-bit, atten 11 dB
  // Filtro paso bajo digital para extraer offset DC
  // sqrt(sum(I^2)/N) → I_RMS
```

`config.clampCalibration` (default 22.3) y `config.clampVoltage` (230 V)
escalan el resultado. Se calibra desde la consola con
`clampCalibration <valor>` mientras se observa la corriente real con una
pinza externa.

## 5. Triac / Dimmer

Dos modos:

1. **Estándar**: rango PWM 0..1023 (10 bits LEDC) sobre triac
   convencional. Frecuencia útil: 100, 50, 25, 12.5 Hz para zero-cross
   o 30 kHz para dimmers semi-conmutados.
2. **Low-cost** (`config.flags.dimmerLowCost`): el módulo chino tiene
   una zona muerta inicial (no entrega potencia entre 0 y ~20 %).
   FreeDS limita `OutputLimits = (209..maxPwmLowCost)` y mapea el
   porcentaje real al rango efectivo.

## 6. Relés (4 salidas auxiliares)

Conmutables por dos criterios independientes:

- **Por % PWM**: cuando `pwm.pwmValue >= RXMin` se enciende; al bajar a
  `RXMin - 10` se apaga.
- **Por watts intercambiados con red**: cuando `wgrid > RXPotOn` se
  enciende; al bajar de `RXPotOff` se apaga (con histéresis).

Cada relé tiene además un override **manual** (`Flags.RelayXMan` /
`config.relaysFlags.RXMan`) que lo fuerza ON.

## 7. Sensores DS18B20

Hasta 15 sensores enumerados al boot (`buildSensorArray()`). Tres roles
asignables:
- `termoSensorAddress`: temperatura del termo (corte de seguridad).
- `triacSensorAddress`: temperatura del disipador del triac.
- `customSensorAddress`: cualquier otro punto.

`config.modoTemperatura`:

| Valor | Comportamiento                                 |
|-------|------------------------------------------------|
| 0     | Sensores informativos, sin actuar              |
| 1     | Apaga PWM en modo AUTO al superar `Apagado`    |
| 2     | Apaga PWM en modo MANUAL al superar `Apagado`  |
| 3     | Apaga PWM en cualquier modo                    |

## 8. Captive portal y AP fallback

Si `config.flags.wifi == false`, el firmware crea AP **`FreeDS`**
(192.168.4.1) con DNS catch-all y formulario inline para configurar la
red (clase `CaptiveRequestHandler` en `FreeDS.ino:606-654`). Tras
guardar, reinicia y entra en modo STA.
