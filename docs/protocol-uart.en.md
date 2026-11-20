# UART protocol STM32 ↔ Smart module (EGA1800)

*Español: [protocolo-uart.md](protocolo-uart.md)*

This is the protocol the ESPHome firmware speaks with the board's STM32.
Understanding it helps adapt the YAML to another gate variant, or debug issues.
All of this was obtained by reverse engineering (the STM32 and the stock ESP
firmware) and **verified live** on a real EGA1800.

## Physical layer

- **115200 baud, 8N1** (8 data, 1 stop, no parity).
- **Variable-length** frames, delimited by an **IDLE** line (a gap of silence
  marks the end of a frame). There is no length byte or checksum at the STM32
  layer.
- The ESP is the **master**: the STM32 stays silent until it is polled.

## Wiring (Smart-module pins, i.e. the ESP32-C2)

| Module pin | Signal      | ESP32-C2 | STM32            | Direction               |
|------------|-------------|----------|------------------|-------------------------|
| 1          | VCC 3.3 V   | —        | —                | power                   |
| 2          | GND         | —        | —                | —                       |
| 3          | data        | GPIO7    | PA9 (USART1_TX)  | STM32 → ESP (state)     |
| 4          | data        | GPIO6    | PA10 (USART1_RX) | ESP → STM32 (commands)  |
| 5          | "Smart" LED | GPIO10   | —                | LED output (active-low) |

In ESPHome: `tx_pin: GPIO6`, `rx_pin: GPIO7`.

## Frame format

```
55 AA <type> [payload...]
```

Fixed header `0x55 0xAA` + a **type** byte. No checksum.

### Commands (ESP → STM32)

| Frame                     | Meaning                                       |
|---------------------------|-----------------------------------------------|
| `55 AA 2E`                | Ask for model/firmware (replies with JSON)    |
| `55 AA 32`                | Poll state (replies with a type-32 frame)     |
| `55 AA 01 <channel> 02`   | **Control press** (step by step)              |

**Control is STEP BY STEP**, just like the physical remote. Each
`55 AA 01 <channel> 02` emulates **one press**, and the STM32 cycles internally:
`open → stop → close → stop → ...`. There are no separate discrete
open/close/stop commands.

| Channel | Moves                         |
|---------|-------------------------------|
| `01`    | Pedestrian (one leaf)         |
| `02`    | Full gate (both leaves)       |
| `03` / `04` | Not programmed (just a beep) |

> The trailing `02` byte is required: `55 AA 01 01` (without it) is **ACKed by the
> STM32 but does not move** the motor.

### Model/firmware reply (STM32 → ESP)

To `55 AA 2E` the STM32 replies with ASCII JSON after the header, e.g.:

```
55 AA 2E {"type":"EGA1800","version":"1.3.0"}
```

### State frame (STM32 → ESP)

Arrives as a reply to the poll (`type 0x32`) or as a spontaneous event when state
changes (`type 0x41`). It is **19 bytes**, indexed from 0. Two-leaf gate (A and B):

```
idx:  0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18
      55 AA TT 00 SA SB 00 00 Bh Bl 00 00 00 01 PR PA PB CC 0D
```

| Byte(s)  | Field                        | Decoding                                      |
|----------|------------------------------|-----------------------------------------------|
| `[0..1]` | Header                       | fixed `55 AA`                                  |
| `[2]`    | Type                         | `0x32` (poll) / `0x41` (event)                 |
| `[4]`    | Leaf A state                 | see state table                               |
| `[5]`    | Leaf B state                 | see state table                               |
| `[8..9]` | Battery (mV, big-endian)     | `(b8<<8 \| b9) / 1000` → volts                 |
| `[14]`   | Backup battery present       | `0`=absent, `≠0`=present                        |
| `[15]`   | Leaf A position %            | `0`=closed … `100`=open                         |
| `[16]`   | Leaf B position %            | `0`=closed … `100`=open                         |
| `[17]`   | Counter (?)                  | varies; unused                                 |

**State codes** (`[4]` and `[5]`):

| Value  | Meaning           |
|--------|-------------------|
| `0x01` | opening           |
| `0x00` | closing           |
| `0x02` | open (at rest)    |
| `0x03` | closed (at rest)  |

The cover position in Home Assistant is the **average of both leaves**
(`(PA + PB) / 200`). It is marked opening/closing if **any** leaf is, with a
position guard so it doesn't get "stuck" at the end stop.

#### Real captured examples

```
55 AA 32 00 03 03 ... 01 00 00 0B 0D   A/B closed,  leaves 0 / 0
55 AA 32 00 03 01 ... 01 00 54 0B 0D   pedestrian opening (leaf B at 84%)
55 AA 32 00 01 01 ... 01 4B 4F 0C 0D   full opening (leaves 75% / 79%)
55 AA 32 00 02 02 ... 01 64 64 0B 0D   A/B open,    leaves 100 / 100
55 AA 32 00 00 00 ... 01 32 49 0B 0D   full closing (leaves 50% / 73%)
```

## Notes for other variants (EGA18, EGB18, EGA15, EGB1900…)

The state-frame layout may **differ** between models (the sibling EGB-18 firmware,
for example, does not use these same offsets). If you convert another model:

1. Set `logger: DEBUG` and keep the parser's `UARTDebug::log_hex` active.
2. Move the gate with the physical remote and capture the `55 AA 32` / `55 AA 41`
   frames.
3. Adjust the byte indices (battery, position, state) to match what you see.

The `55 AA 01 <channel> 02` control command was the most consistent across
variants, but it's worth confirming live too.
