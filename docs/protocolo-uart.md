# Protocolo UART STM32 ↔ módulo Smart (EGA1800)

Este es el protocolo que habla el firmware de ESPHome con el STM32 de la placa.
Entenderlo sirve para adaptar el YAML a otra variante del portón o para depurar.
Todo esto se obtuvo por ingeniería inversa (firmware del STM32 y del ESP stock) y
se **verificó en vivo** sobre un EGA1800 real.

## Capa física

- **115200 baud, 8N1** (8 data, 1 stop, sin paridad).
- Tramas de **largo variable**, delimitadas por línea **IDLE** (un gap de silencio
  marca el fin de trama). No hay byte de largo ni checksum en la capa STM32.
- El ESP es el **master**: el STM32 queda mudo hasta que se lo sondea.

## Conexión (pines del módulo Smart, que es el ESP32-C2)

| Pin módulo | Señal        | ESP32-C2 | STM32           | Dirección               |
|------------|--------------|----------|-----------------|-------------------------|
| 1          | VCC 3.3 V    | —        | —               | alimentación            |
| 2          | GND          | —        | —               | —                       |
| 3          | datos        | GPIO7    | PA9 (USART1_TX) | STM32 → ESP (estado)    |
| 4          | datos        | GPIO6    | PA10 (USART1_RX)| ESP → STM32 (comandos)  |
| 5          | LED "Smart"  | GPIO10   | —               | salida LED (activo-bajo)|

En ESPHome: `tx_pin: GPIO6`, `rx_pin: GPIO7`.

## Formato de trama

```
55 AA <tipo> [payload...]
```

Cabecera fija `0x55 0xAA` + un byte de **tipo**. Sin checksum.

### Comandos (ESP → STM32)

| Trama                     | Significado                                   |
|---------------------------|-----------------------------------------------|
| `55 AA 2E`                | Pedir modelo/firmware (responde JSON)         |
| `55 AA 32`                | Poll de estado (responde trama tipo 32)       |
| `55 AA 01 <canal> 02`     | **Pulsación de control** (paso a paso)        |

**El control es PASO A PASO**, igual que el control remoto físico. Cada
`55 AA 01 <canal> 02` emula **una pulsación** y el STM32 cicla internamente:
`abrir → parar → cerrar → parar → ...`. No existen comandos discretos
separados de abrir/cerrar/parar.

| Canal | Mueve                          |
|-------|--------------------------------|
| `01`  | Peatonal (una hoja)            |
| `02`  | Portón completo (ambas hojas)  |
| `03` / `04` | Sin programar (sólo un beep) |

> El byte final `02` es necesario: `55 AA 01 01` (sin él) el STM32 lo **ACKea
> pero no mueve** el motor.

### Respuesta de modelo/firmware (STM32 → ESP)

A `55 AA 2E` el STM32 responde JSON ASCII tras el header, p. ej.:

```
55 AA 2E {"type":"EGA1800","version":"1.3.0"}
```

### Trama de estado (STM32 → ESP)

Llega como respuesta al poll (`tipo 0x32`) o como evento espontáneo al cambiar
de estado (`tipo 0x41`). Son **19 bytes**, indexados desde 0. Portón de dos
hojas (A y B):

```
idx:  0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18
      55 AA TT 00 SA SB 00 00 Bh Bl 00 00 00 01 PR PA PB CC 0D
```

| Byte(s)  | Campo                       | Decodificación                                   |
|----------|-----------------------------|--------------------------------------------------|
| `[0..1]` | Cabecera                    | fija `55 AA`                                      |
| `[2]`    | Tipo                        | `0x32` (poll) / `0x41` (evento)                   |
| `[4]`    | Estado hoja A               | ver tabla de estados                             |
| `[5]`    | Estado hoja B               | ver tabla de estados                             |
| `[8..9]` | Batería (mV, big-endian)    | `(b8<<8 \| b9) / 1000` → voltios                  |
| `[14]`   | Batería de respaldo presente| `0`=ausente, `≠0`=presente                        |
| `[15]`   | % posición hoja A           | `0`=cerrada … `100`=abierta                       |
| `[16]`   | % posición hoja B           | `0`=cerrada … `100`=abierta                       |
| `[17]`   | Contador (?)                | varía; no se usa                                  |

**Códigos de estado** (`[4]` y `[5]`):

| Valor  | Significado         |
|--------|---------------------|
| `0x01` | abriendo            |
| `0x00` | cerrando            |
| `0x02` | abierto (en reposo) |
| `0x03` | cerrado (en reposo) |

La posición del cover en Home Assistant es el **promedio de ambas hojas**
(`(PA + PB) / 200`). Se marca abriendo/cerrando si **alguna** hoja lo está,
con guarda de posición para no quedar "pegado" al tope.

#### Ejemplos reales capturados

```
55 AA 32 00 03 03 ... 01 00 00 0B 0D   A/B cerrado,  hojas 0 / 0
55 AA 32 00 03 01 ... 01 00 54 0B 0D   peatonal abriendo (hoja B al 84%)
55 AA 32 00 01 01 ... 01 4B 4F 0C 0D   completo abriendo (hojas 75% / 79%)
55 AA 32 00 02 02 ... 01 64 64 0B 0D   A/B abierto,  hojas 100 / 100
55 AA 32 00 00 00 ... 01 32 49 0B 0D   completo cerrando (hojas 50% / 73%)
```

## Notas para otras variantes (EGA18, EGB18, EGA15, EGB1900…)

El layout de la trama de estado puede **diferir** entre modelos (el firmware
hermano EGB-18, por ejemplo, no usa estos mismos offsets). Si convertís otro
modelo:

1. Poné `logger: DEBUG` y dejá activo el `UARTDebug::log_hex` del parser.
2. Mové el portón con el control físico y capturá las tramas `55 AA 32` / `55 AA 41`.
3. Ajustá los índices de bytes (batería, posición, estado) a lo que veas.

El comando de control `55 AA 01 <canal> 02` fue lo más consistente entre
variantes, pero conviene confirmarlo también en vivo.
