# LiftLine Serial Protocol Documentation

## Overview

This document describes the complete serial communication protocol for the LiftLine hoverboard variant. The protocol is based on the original EFeru/hoverboard-firmware-hack-FOC project and uses USART2/USART3 at 38400 baud for bidirectional communication.

**Protocol Status:** ✅ Fully conformant with EFeru/hoverboard-firmware-hack-FOC

---

## 📡 Communication Configuration

| Parameter | Value |
|-----------|-------|
| **Baud Rate** | 38400 bps |
| **Data Bits** | 8 |
| **Parity** | None |
| **Stop Bits** | 1 |
| **Format** | 8N1 |
| **Byte Order** | Little-Endian (LSB first) |
| **UART Interfaces** | USART2, USART3 |
| **Transmission Mode** | DMA (Direct Memory Access) |
| **Update Frequency** | 50 Hz (every 20 ms) |

---

## 📤 Transmission: SerialFeedback (26 bytes)

### Structure Definition

```c
typedef struct{
  uint16_t  start;           // [0-1]   0xABCD (SERIAL_START_FRAME)
  int16_t   cmd1;            // [2-3]   Commande input1 brute [-1000, 1000]
  int16_t   cmd2;            // [4-5]   Commande input2 brute [-1000, 1000]
  int16_t   speedR_meas;     // [6-7]   Vitesse droite en km/h × 100
  int16_t   speedL_meas;     // [8-9]   Vitesse gauche en km/h × 100
  int16_t   batVoltage;      // [10-11] Tension batterie en V × 100
  int16_t   boardTemp;       // [12-13] Température carte en °C × 10
  int16_t   posR_cm;         // [14-15] Position droite en décimètres (dm)
  int16_t   posL_cm;         // [16-17] Position gauche en décimètres (dm)
  int16_t   dcCurrL;         // [18-19] Courant DC gauche en A × 100
  int16_t   dcCurrR;         // [20-21] Courant DC droit en A × 100
  uint16_t  statusFlags;     // [22-23] LSB=errorFlags, MSB=statusByte
  uint16_t  checksum;        // [24-25] XOR de tous les champs
} SerialFeedback;  // Total: 26 octets
```

### Field Description

#### Basic Commands
| Field | Type | Range | Unit | Description |
|-------|------|-------|------|-------------|
| `start` | uint16_t | `0xABCD` | - | Frame start identifier (mandatory) |
| `cmd1` | int16_t | -1000 to +1000 | - | Input 1 command (raw value) |
| `cmd2` | int16_t | -1000 to +1000 | - | Input 2 command (raw value) |

#### Motor Telemetry
| Field | Type | Scaling | Unit | Range | Resolution |
|-------|------|---------|------|-------|------------|
| `speedR_meas` | int16_t | ÷100 | km/h | ±327.67 | 0.01 km/h |
| `speedL_meas` | int16_t | ÷100 | km/h | ±327.67 | 0.01 km/h |
| `posR_cm` | int16_t | ÷10 | m | ±3276.7 | 0.1 m (10 cm) |
| `posL_cm` | int16_t | ÷10 | m | ±3276.7 | 0.1 m (10 cm) |

#### Power & Temperature
| Field | Type | Scaling | Unit | Range | Resolution |
|-------|------|---------|------|-------|------------|
| `batVoltage` | int16_t | ÷100 | V | ±327.67 | 0.01 V (10 mV) |
| `boardTemp` | int16_t | ÷10 | °C | ±3276.7 | 0.1 °C |
| `dcCurrL` | int16_t | ÷100 | A | ±327.67 | 0.01 A (10 mA) |
| `dcCurrR` | int16_t | ÷100 | A | ±327.67 | 0.01 A (10 mA) |

#### Status & Diagnostics
| Field | Type | Description |
|-------|------|-------------|
| `statusFlags` | uint16_t | Combined error and status flags (see below) |
| `checksum` | uint16_t | XOR checksum for frame validation |

---

## 🚦 Status Flags (statusFlags - uint16_t)

The `statusFlags` field combines two 8-bit values in little-endian format:
- **LSB (Byte 22):** errorFlags - Hardware/system errors
- **MSB (Byte 23):** statusByte - System status and operational modes

### Extraction in Receiver Code

```c
// Extract individual flags
uint8_t errorFlags = statusFlags & 0xFF;        // LSB
uint8_t statusByte = (statusFlags >> 8) & 0xFF; // MSB
```

### Error Flags (LSB) - Bit-wise Definition

```c
#define ERROR_FLAG_MOTOR_LEFT       (1 << 0)  // 0x01
#define ERROR_FLAG_MOTOR_RIGHT      (1 << 1)  // 0x02
#define ERROR_FLAG_TIMEOUT_SERIAL   (1 << 2)  // 0x04
#define ERROR_FLAG_TIMEOUT_ADC      (1 << 3)  // 0x08
#define ERROR_FLAG_BAT_LOW          (1 << 4)  // 0x10
#define ERROR_FLAG_BAT_CRITICAL     (1 << 5)  // 0x20
#define ERROR_FLAG_TEMP_HIGH        (1 << 6)  // 0x40
#define ERROR_FLAG_MOTOR_DISABLED   (1 << 7)  // 0x80
```

| Bit | Mask | Name | Condition |
|-----|------|------|-----------|
| 0 | 0x01 | MOTOR_LEFT | Left FOC controller error |
| 1 | 0x02 | MOTOR_RIGHT | Right FOC controller error |
| 2 | 0x04 | TIMEOUT_SERIAL | No command received for 0.8s |
| 3 | 0x08 | TIMEOUT_ADC | ADC sensor timeout |
| 4 | 0x10 | BAT_LOW | Battery < BAT_LVL1 (35.0V) |
| 5 | 0x20 | BAT_CRITICAL | Battery < BAT_DEAD (33.7V) |
| 6 | 0x40 | TEMP_HIGH | Temperature > 60°C (600 in ×10) |
| 7 | 0x80 | MOTOR_DISABLED | Motors disabled (enable = 0) |

### Status Byte (MSB) - Bit-wise Definition

```c
#define STATUS_ENABLE               (1 << 0)  // 0x01
#define STATUS_BACKWARD             (1 << 6)  // 0x40
#define STATUS_BRAKE_ACTIVE         (1 << 7)  // 0x80
#define BAT_LEVEL_MASK              0x0E      // bits 1-3
#define CTRL_MODE_MASK              0x30      // bits 4-5
```

| Bit | Mask | Name | Values/Description |
|-----|------|------|-------------------|
| 0 | 0x01 | ENABLE | 1=Motors ON, 0=Motors OFF |
| 1-3 | 0x0E | BAT_LEVEL | 0-7 (see battery levels below) |
| 4-5 | 0x30 | CTRL_MODE | 0=OPEN, 1=VOLTAGE, 2=SPEED, 3=TORQUE |
| 6 | 0x40 | BACKWARD | 1=Backward drive active |
| 7 | 0x80 | BRAKE_ACTIVE | 1=Braking detected |

### Battery Level Extraction & Values

```c
// Extract battery level from status byte
uint8_t batLevel = (statusByte & BAT_LEVEL_MASK) >> 1;

// Battery level meanings (see config.h definitions)
```

| Level | Voltage | Status | Description |
|-------|---------|--------|-------------|
| 0 | < 33.7V | 🔴 CRITICAL | Battery nearly dead |
| 1 | 33.7-35.0V | 🔴 VERY LOW | Critical threshold |
| 2 | 35.0-36.0V | 🔴 LOW | Warning threshold |
| 3 | 36.0-37.0V | 🟡 MEDIUM-LOW | Low capacity |
| 4 | 37.0-38.0V | 🟡 MEDIUM | Normal operation |
| 5 | 38.0-39.0V | 🟢 GOOD | Good capacity |
| 6-7 | > 39.0V | 🟢 FULL | Full charge |

### Control Mode Extraction

```c
// Extract control mode from status byte
uint8_t ctrlMode = (statusByte & CTRL_MODE_MASK) >> 4;

// Control mode meanings
```

| Mode | Value | Description |
|------|-------|-------------|
| OPEN | 0 | Open loop (direct PWM) |
| VOLTAGE | 1 | Voltage mode control |
| SPEED | 2 | Speed (RPM) control |
| TORQUE | 3 | Torque (current) control |

---

## ✅ Checksum Calculation & Validation

### Transmission (STM32 Firmware)

The checksum is calculated by XORing each field **individually** as uint16_t values:

```c
// VARIANT_LIFTLINE checksum calculation
Feedback.checksum = (uint16_t)(
    Feedback.start 
    ^ Feedback.cmd1 
    ^ Feedback.cmd2 
    ^ Feedback.speedR_meas 
    ^ Feedback.speedL_meas 
    ^ Feedback.batVoltage 
    ^ Feedback.boardTemp 
    ^ Feedback.posR_cm 
    ^ Feedback.posL_cm 
    ^ Feedback.dcCurrL 
    ^ Feedback.dcCurrR 
    ^ (uint16_t)errorFlags 
    ^ (uint16_t)statusByte
);
```

### Reception & Validation (Receiver Side)

```c
// Calculate checksum with received data
uint16_t calculated_checksum = 
    feedback.start 
    ^ feedback.cmd1 
    ^ feedback.cmd2 
    ^ feedback.speedR_meas 
    ^ feedback.speedL_meas 
    ^ feedback.batVoltage 
    ^ feedback.boardTemp 
    ^ feedback.posR_cm 
    ^ feedback.posL_cm 
    ^ feedback.dcCurrL 
    ^ feedback.dcCurrR 
    ^ (uint16_t)(feedback.statusFlags & 0xFF)        // errorFlags
    ^ (uint16_t)((feedback.statusFlags >> 8) & 0xFF); // statusByte

// Validate frame
if (calculated_checksum != feedback.checksum) {
    // Checksum mismatch - frame corrupted
    return ERROR_CHECKSUM_INVALID;
}

// Validate start frame
if (feedback.start != 0xABCD) {
    // Invalid start frame
    return ERROR_START_FRAME_INVALID;
}

// Frame is valid - process data
return SUCCESS;
```

### Important Notes

1. **Individual Field XOR:** Each field is XORed separately, not combined into bytes first
2. **Type Casting:** errorFlags and statusByte are cast to uint16_t before XORing
3. **Byte Order:** Little-endian transmission - LSB sent first
4. **No Padding:** All fields are 2-byte aligned (uint16_t or int16_t only)

---

## 🔄 Little-Endian Byte Order

The protocol uses little-endian byte transmission (LSB first). Example:

```
Example: speedR_meas = 1234 (0x04D2 in hex)
  Byte [6] = 0xD2  (LSB - sent first)
  Byte [7] = 0x04  (MSB - sent second)

Example: statusFlags = 0x4201 (errorFlags=0x01, statusByte=0x42)
  Byte [22] = 0x01  (errorFlags - LSB sent first)
  Byte [23] = 0x42  (statusByte - MSB sent second)
```

When receiving, reconstruct uint16_t values as:
```
value_16bit = byte_0 | (byte_1 << 8);
```

---

## 📥 Reception: SerialCommand (8 bytes)

The firmware receives command frames on USART3. This is the simpler reception structure:

```c
typedef struct {
    uint16_t start;     // [0-1] 0xABCD
    int16_t speedL;     // [2-3] Left motor speed [-1000, +1000]
    int16_t speedR;     // [4-5] Right motor speed [-1000, +1000]
    uint16_t checksum;  // [6-7] XOR: start ^ speedL ^ speedR
} SerialCommand;
```

### Reception Validation

```c
// Validation in util.c (lines 1282-1288)
if (command_in->start == SERIAL_START_FRAME) {
    #if defined(VARIANT_LIFTLINE)
    checksum = (uint16_t)(
        command_in->start 
        ^ command_in->speedL 
        ^ command_in->speedR
    );
    #endif
    
    if (command_in->checksum == checksum) {
        // Valid command - execute it
        *command_out = *command_in;
        return SUCCESS;
    }
}
return ERROR_INVALID_CHECKSUM;
```

---

## 📊 Practical Examples

### Example 1: Decode Received Feedback Frame

**Raw bytes received (26 bytes):**
```
CD AB  // start = 0xABCD
64 00  // cmd1 = 100
C8 00  // cmd2 = 200
6A 0B  // speedR_meas = 2922 (29.22 km/h)
64 0B  // speedL_meas = 2916 (29.16 km/h)
CE 0E  // batVoltage = 3790 (37.90 V)
C6 00  // boardTemp = 198 (19.8°C)
78 00  // posR_cm = 120 (1.2 m)
7C 00  // posL_cm = 124 (1.24 m)
F4 01  // dcCurrL = 500 (5.00 A)
08 02  // dcCurrR = 520 (5.20 A)
01 42  // statusFlags = 0x4201 (errorFlags=0x01, statusByte=0x42)
15 AF  // checksum = 0xAF15
```

**Python Decoding:**
```python
import struct

data = bytes.fromhex('CDAB6400C80006A0B640B6E0EC6000C678007C00F40108025201AF15')
unpacked = struct.unpack('<Hhhhhhhhhhhh HH', data)

feedback = {
    'start': unpacked[0],              # 0xABCD
    'cmd1': unpacked[1],               # 100
    'cmd2': unpacked[2],               # 200
    'speedR_kmh': unpacked[3] / 100,   # 29.22
    'speedL_kmh': unpacked[4] / 100,   # 29.16
    'batVoltage_V': unpacked[5] / 100, # 37.90
    'boardTemp_C': unpacked[6] / 10,   # 19.8
    'posR_m': unpacked[7] / 10,        # 1.2
    'posL_m': unpacked[8] / 10,        # 1.24
    'dcCurrL_A': unpacked[9] / 100,    # 5.00
    'dcCurrR_A': unpacked[10] / 100,   # 5.20
    'statusFlags': unpacked[11],       # 0x4201
    'checksum': unpacked[12]           # 0xAF15
}

# Extract status
errorFlags = feedback['statusFlags'] & 0xFF           # 0x01
statusByte = (feedback['statusFlags'] >> 8) & 0xFF   # 0x42

# Battery level (bits 1-3)
batLevel = (statusByte & 0x0E) >> 1  # 1 = Very Low

# Control mode (bits 4-5)
ctrlMode = (statusByte & 0x30) >> 4  # 2 = Speed mode
```

### Example 2: Validate Checksum

```python
def validate_liftline_feedback(data):
    """Validate LiftLine feedback frame (26 bytes)"""
    
    if len(data) != 26:
        return False, "Invalid frame length"
    
    # Unpack little-endian
    unpacked = struct.unpack('<Hhhhhhhhhhhh HH', data)
    
    # Check start frame
    if unpacked[0] != 0xABCD:
        return False, "Invalid start frame"
    
    # Calculate checksum
    calc_checksum = unpacked[0]
    for i in range(1, 11):
        calc_checksum ^= unpacked[i]
    
    status_flags = unpacked[11]
    calc_checksum ^= (status_flags & 0xFF)           # errorFlags
    calc_checksum ^= ((status_flags >> 8) & 0xFF)   # statusByte
    
    # Validate
    if calc_checksum != unpacked[12]:
        return False, f"Checksum mismatch"
    
    return True, "Frame valid"
```

---

## 📝 Implementation Checklist

- [ ] Validate **start frame** (must be 0xABCD)
- [ ] Verify **checksum** before processing data
- [ ] Extract **errorFlags** from LSB of statusFlags
- [ ] Extract **statusByte** from MSB of statusFlags
- [ ] Parse **battery level** from bits 1-3 of statusByte
- [ ] Parse **control mode** from bits 4-5 of statusByte
- [ ] Convert **speed values** (÷100 for km/h)
- [ ] Convert **position values** (÷10 for meters)
- [ ] Convert **voltage** (÷100 for volts)
- [ ] Convert **temperature** (÷10 for °C)
- [ ] Convert **current** (÷100 for amps)
- [ ] Handle **little-endian** byte order correctly
- [ ] Implement **timeout handling** (>0.8s without command)

---

## 🔗 References

- **Original Project:** [EFeru/hoverboard-firmware-hack-FOC](https://github.com/EFeru/hoverboard-firmware-hack-FOC)
- **Arduino Example:** See `Arduino/hoverserial/hoverserial.ino` in original project
- **Protocol Conformance:** 100% compatible with EFeru serial protocol

---

## 📝 Notes

- **Structure Alignment:** All fields are 2-byte aligned (uint16_t/int16_t only) - no padding
- **Transmission Frequency:** 50 Hz (20 ms interval)
- **Timeout Protection:** Commands timeout after 0.8 seconds without update
- **Error Handling:** Always validate checksum before processing data
- **Endianness:** Little-endian throughout - LSB transmitted first

## Feedback Output (Telemetry Frame)
**Direction:** Hoverboard → Controller  
**Port:** USART2 or USART3  

---

## ⏱️ Timing & Real‑Time Impact

### Line Occupancy at 38400 bps
- UART sends 1 start bit + 8 data bits + 1 stop bit → 10 bits/byte
- Wire time ≈ `bytes × 10 / baud`

| Frame | Bytes | Bits | Time |
|-------|-------|------|------|
| Feedback (Telemetry) | 26 | 260 | ~6.77 ms |
| Command (Control)     | 8  | 80  | ~2.08 ms |

### Update Cadence vs Wire Time
- Feedback frames are issued every 20 ms (50 Hz)
- At 38400 bps, a 26‑byte frame occupies the line ~6.77 ms
- Remaining ~13.23 ms per 20 ms period is idle on that UART line

### CPU Load & Determinism
- **Transmission uses DMA** (`HAL_UART_Transmit_DMA`) → non‑blocking; CPU schedules DMA and returns
- **Reception uses DMA + memcpy + lightweight XOR** → short, bounded processing in main/utility code
- No busy‑waits on UART in the shown implementation; control loop remains deterministic

### Dual UART Considerations
- USART2 (feedback TX) and USART3 (command RX) are independent peripherals
- Their wire times do not interfere; DMA engines operate concurrently
- Ensure NVIC priorities keep motor control and EXTI (Hall) above UART IRQs if needed

### Practical Recommendations
- Keep baud ≥ 38400 for 26‑byte @100 Hz; higher baud (57600/115200) further reduces occupancy
- Avoid blocking `HAL_UART_Transmit` in control paths; stick to DMA
- Validate `sizeof(SerialFeedback) == 26` to prevent padding regressions
- If adding fields, re‑evaluate `bytes × 10 / baud` to keep occupancy < update period

### Quick Formula
```
time_ms ≈ (frame_bytes × 10 × 1000) / baud
```
Example: `(26 × 10 × 1000) / 38400 ≈ 6.77 ms`
**Size:** 28 bytes  
**Rate:** Every 10ms (100 Hz)

### Structure
```c
typedef struct {
    uint16_t start;         // Start frame: 0xABCD
    int16_t  cmd1;          // Echo of input command 1
    int16_t  cmd2;          // Echo of input command 2
    int16_t  speedR_meas;   // Right motor speed (km/h × 100)
    int16_t  speedL_meas;   // Left motor speed (km/h × 100)
    int16_t  batVoltage;    // Battery voltage (V × 100)
    int16_t  boardTemp;     // Board temperature (°C × 10)
    int16_t  posR_cm;       // Right position (decimeters)
    int16_t  posL_cm;       // Left position (decimeters)
    int16_t  dcCurrR;       // Right DC current (A × 100)
    int16_t  dcCurrL;       // Left DC current (A × 100)
    uint8_t  errorFlags;    // Error flags byte
    uint8_t  statusByte;    // Status byte
    uint16_t checksum;      // XOR checksum
} SerialFeedback;
```

### Feedback Frame Format
| Offset | Field | Type | Unit | Range | Description |
|--------|-------|------|------|-------|-------------|
| 0 | `start` | uint16 | - | 0xABCD | Frame identifier |
| 2 | `cmd1` | int16 | - | -1000 to +1000 | Command echo |
| 4 | `cmd2` | int16 | - | -1000 to +1000 | Command echo |
| 6 | `speedR_meas` | int16 | km/h×100 | -6000 to +6000 | Right speed |
| 8 | `speedL_meas` | int16 | km/h×100 | -6000 to +6000 | Left speed |
| 10 | `batVoltage` | int16 | V×100 | 0 to 6000 | Battery voltage |
| 12 | `boardTemp` | int16 | °C×10 | -500 to +1500 | Temperature |
| 14 | `posR_cm` | int16 | dm | -32768 to +32767 | Right position |
| 16 | `posL_cm` | int16 | dm | -32768 to +32767 | Left position |
| 18 | `dcCurrR` | int16 | A×100 | -10000 to +10000 | Right current |
| 20 | `dcCurrL` | int16 | A×100 | -10000 to +10000 | Left current |
| 22 | `errorFlags` | uint8 | bits | 0x00 to 0xFF | Error flags |
| 23 | `statusByte` | uint8 | bits | 0x00 to 0xFF | Status byte |
| 24 | `checksum` | uint16 | - | - | XOR checksum |

### Checksum Calculation
```c
checksum = start ^ cmd1 ^ cmd2 ^ speedR_meas ^ speedL_meas 
         ^ batVoltage ^ boardTemp ^ posR_cm ^ posL_cm 
         ^ dcCurrR ^ dcCurrL 
         ^ ((uint16_t)errorFlags | ((uint16_t)statusByte << 8))
```

### Example Feedback Frame (Hex)
```
CD AB 00 00 00 00 00 00 00 00 D6 0E C6 00 00 00 00 00 FE FF 00 00 00 04 68 0F
```
Breakdown:
- `CD AB` = start (0xABCD)
- `00 00` = cmd1 (0)
- `00 00` = cmd2 (0)
- `00 00` = speedR_meas (0 = 0.00 km/h)
- `00 00` = speedL_meas (0 = 0.00 km/h)
- `D6 0E` = batVoltage (0x0ED6 = 3798 = 37.98 V)
- `C6 00` = boardTemp (0x00C6 = 198 = 19.8°C)
- `00 00` = posR_cm (0 dm = 0.0 m)
- `00 00` = posL_cm (0 dm = 0.0 m)
- `FE FF` = dcCurrR (0xFFFE = -2 = -0.02 A)
- `00 00` = dcCurrL (0 = 0.00 A)
- `00` = errorFlags (0x00 = no errors)
- `04` = statusByte (0x04 = motors enabled, SPD mode)
- `68 0F` = checksum

## Error Flags Byte (Offset 22)

| Bit | Mask | Name | Description |
|-----|------|------|-------------|
| 0 | 0x01 | `ERROR_FLAG_MOTOR_LEFT` | Left motor FOC error detected |
| 1 | 0x02 | `ERROR_FLAG_MOTOR_RIGHT` | Right motor FOC error detected |
| 2 | 0x04 | `ERROR_FLAG_TIMEOUT_SERIAL` | Serial timeout (~0.8s no valid command) |
| 3 | 0x08 | `ERROR_FLAG_TIMEOUT_ADC` | ADC sensor timeout detected |
| 4 | 0x10 | `ERROR_FLAG_BAT_LOW` | Battery voltage < 35.0V (BAT_LVL1) |
| 5 | 0x20 | `ERROR_FLAG_BAT_CRITICAL` | Battery voltage < 33.7V (BAT_DEAD) |
| 6 | 0x40 | `ERROR_FLAG_TEMP_HIGH` | Board temperature > 60°C |
| 7 | 0x80 | `ERROR_FLAG_MOTOR_DISABLED` | Motors are disabled (enable = 0) |

### Error Flags Decoding Example (C)
```c
uint8_t errorFlags = feedback.errorFlags;

bool motorLeftError    = (errorFlags & 0x01) != 0;
bool motorRightError   = (errorFlags & 0x02) != 0;
bool serialTimeout     = (errorFlags & 0x04) != 0;
bool adcTimeout        = (errorFlags & 0x08) != 0;
bool batteryLow        = (errorFlags & 0x10) != 0;
bool batteryCritical   = (errorFlags & 0x20) != 0;
bool tempHigh          = (errorFlags & 0x40) != 0;
bool motorsDisabled    = (errorFlags & 0x80) != 0;
```

## Status Byte (Offset 23)

| Bits | Mask | Name | Description |
|------|------|------|-------------|
| 0 | 0x01 | `STATUS_ENABLE` | Motors enabled (1) or disabled (0) |
| 1-3 | 0x0E | `STATUS_BAT_LEVEL` | Battery level (0-7) |
| 4-5 | 0x30 | `STATUS_CTRL_MODE` | Control mode (0-3) |
| 6 | 0x40 | `STATUS_BACKWARD` | Backward drive active |
| 7 | 0x80 | `STATUS_BRAKE_ACTIVE` | Braking detected |

### Battery Level Values (Bits 1-3)
Extract with: `batLevel = (statusByte & 0x0E) >> 1`

| Value | Name | Voltage Range | Indicator |
|-------|------|---------------|-----------|
| 0 | `BAT_LEVEL_CRITICAL` | < 33.7V | Critical |
| 1 | `BAT_LEVEL_LVL1` | 33.7V - 35.0V | 🔴 Red blink |
| 2 | `BAT_LEVEL_LVL2` | 35.0V - 36.0V | 🔴 Red |
| 3 | `BAT_LEVEL_LVL3` | 36.0V - 37.0V | 🟡 Yellow blink |
| 4 | `BAT_LEVEL_LVL4` | 37.0V - 38.0V | 🟡 Yellow |
| 5 | `BAT_LEVEL_LVL5` | 38.0V - 39.0V | 🟢 Green blink |
| 6 | `BAT_LEVEL_FULL` | ≥ 39.0V | 🟢 Green |
| 7 | `BAT_LEVEL_RESERVED` | - | Reserved |

### Control Mode Values (Bits 4-5)
Extract with: `ctrlMode = (statusByte & 0x30) >> 4`

| Value | Mode | Description |
|-------|------|-------------|
| 0 | OPEN_MODE | Open loop control |
| 1 | VLT_MODE | Voltage control mode |
| 2 | SPD_MODE | Speed control mode (default for VARIANT_LIFTLINE) |
| 3 | TRQ_MODE | Torque control mode |

### Status Byte Decoding Example (C)
```c
uint8_t statusByte = feedback.statusByte;

bool motorsEnabled = (statusByte & 0x01) != 0;
uint8_t batLevel   = (statusByte & 0x0E) >> 1;  // 0-7
uint8_t ctrlMode   = (statusByte & 0x30) >> 4;  // 0-3
bool backward      = (statusByte & 0x40) != 0;
bool braking       = (statusByte & 0x80) != 0;

// Interpret battery level
const char* batLevelStr[] = {
    "Critical", "Very Low", "Low", "Medium-Low",
    "Medium", "Good", "Full", "Reserved"
};
printf("Battery: %s\n", batLevelStr[batLevel]);

// Interpret control mode
const char* ctrlModeStr[] = {"OPEN", "VLT", "SPD", "TRQ"};
printf("Mode: %s\n", ctrlModeStr[ctrlMode]);
```

## Data Conversion Examples

### Speed Conversion
```c
// Firmware to km/h
float speedKmh = (float)feedback.speedR_meas / 100.0f;  // 2922 → 29.22 km/h

// km/h to feedback value
int16_t speedValue = (int16_t)(speedKmh * 100.0f);  // 29.22 → 2922
```

### Voltage Conversion
```c
// Firmware to Volts
float voltage = (float)feedback.batVoltage / 100.0f;  // 3798 → 37.98 V

// Volts to feedback value
int16_t voltageValue = (int16_t)(voltage * 100.0f);  // 37.98 → 3798
```

### Temperature Conversion
```c
// Firmware to Celsius
float tempC = (float)feedback.boardTemp / 10.0f;  // 198 → 19.8°C

// Celsius to feedback value
int16_t tempValue = (int16_t)(tempC * 10.0f);  // 19.8 → 198
```

### Position Conversion
```c
// Firmware to meters
float distanceM = (float)feedback.posR_cm / 10.0f;  // 1234 → 123.4 m

// Meters to feedback value
int16_t posValue = (int16_t)(distanceM * 10.0f);  // 123.4 → 1234
```

### Current Conversion
```c
// Firmware to Amperes
float currentA = (float)feedback.dcCurrR / 100.0f;  // 500 → 5.00 A

// Amperes to feedback value
int16_t currentValue = (int16_t)(currentA * 100.0f);  // 5.00 → 500
```

## Python Implementation Example

```python
import struct

class HoverboardFeedback:
    def __init__(self, data):
        # Unpack little-endian data
        unpacked = struct.unpack('<Hhhhhhhhhhhbbh', data)
        
        self.start = unpacked[0]          # Should be 0xABCD
        self.cmd1 = unpacked[1]
        self.cmd2 = unpacked[2]
        self.speedR_meas = unpacked[3]
        self.speedL_meas = unpacked[4]
        self.batVoltage = unpacked[5]
        self.boardTemp = unpacked[6]
        self.posR_cm = unpacked[7]
        self.posL_cm = unpacked[8]
        self.dcCurrR = unpacked[9]
        self.dcCurrL = unpacked[10]
        self.errorFlags = unpacked[11] & 0xFF
        self.statusByte = unpacked[12] & 0xFF
        self.checksum = unpacked[13]
        
    def validate_checksum(self):
        # Calculate expected checksum
        calc = self.start ^ self.cmd1 ^ self.cmd2
        calc ^= self.speedR_meas ^ self.speedL_meas
        calc ^= self.batVoltage ^ self.boardTemp
        calc ^= self.posR_cm ^ self.posL_cm
        calc ^= self.dcCurrR ^ self.dcCurrL
        calc ^= (self.errorFlags | (self.statusByte << 8))
        return calc == self.checksum
    
    def get_speed_kmh(self):
        return self.speedR_meas / 100.0, self.speedL_meas / 100.0
    
    def get_voltage(self):
        return self.batVoltage / 100.0
    
    def get_temperature(self):
        return self.boardTemp / 10.0
    
    def get_position_m(self):
        return self.posR_cm / 10.0, self.posL_cm / 10.0
    
    def get_current_a(self):
        return self.dcCurrR / 100.0, self.dcCurrL / 100.0
    
    def get_errors(self):
        return {
            'motor_left': bool(self.errorFlags & 0x01),
            'motor_right': bool(self.errorFlags & 0x02),
            'serial_timeout': bool(self.errorFlags & 0x04),
            'adc_timeout': bool(self.errorFlags & 0x08),
            'bat_low': bool(self.errorFlags & 0x10),
            'bat_critical': bool(self.errorFlags & 0x20),
            'temp_high': bool(self.errorFlags & 0x40),
            'motors_disabled': bool(self.errorFlags & 0x80)
        }
    
    def get_status(self):
        return {
            'motors_enabled': bool(self.statusByte & 0x01),
            'battery_level': (self.statusByte & 0x0E) >> 1,
            'control_mode': (self.statusByte & 0x30) >> 4,
            'backward': bool(self.statusByte & 0x40),
            'braking': bool(self.statusByte & 0x80)
        }

class HoverboardCommand:
    @staticmethod
    def create(speedL, speedR):
        start = 0xABCD
        checksum = start ^ speedL ^ speedR
        return struct.pack('<Hhhh', start, speedL, speedR, checksum)

# Usage
command = HoverboardCommand.create(500, 500)  # Both motors at 500
# Send via serial port

# Receive feedback
feedback_data = serial_port.read(28)
feedback = HoverboardFeedback(feedback_data)

if feedback.validate_checksum():
    speedR, speedL = feedback.get_speed_kmh()
    voltage = feedback.get_voltage()
    errors = feedback.get_errors()
    status = feedback.get_status()
    
    print(f"Speed: R={speedR:.2f} km/h, L={speedL:.2f} km/h")
    print(f"Battery: {voltage:.2f} V (Level: {status['battery_level']})")
    print(f"Errors: {errors}")
    print(f"Status: {status}")
```

## C/C++ Implementation Example

```c
#include <stdint.h>
#include <stdbool.h>

typedef struct {
    uint16_t start;         // 0xABCD
    int16_t  speedL;        // Left motor command
    int16_t  speedR;        // Right motor command
    uint16_t checksum;
} __attribute__((packed)) SerialCommand;

typedef struct {
    uint16_t start;         // 0xABCD
    int16_t  cmd1;
    int16_t  cmd2;
    int16_t  speedR_meas;
    int16_t  speedL_meas;
    int16_t  batVoltage;
    int16_t  boardTemp;
    int16_t  posR_cm;
    int16_t  posL_cm;
    int16_t  dcCurrR;
    int16_t  dcCurrL;
    uint8_t  errorFlags;
    uint8_t  statusByte;
    uint16_t checksum;
} __attribute__((packed)) SerialFeedback;

bool validate_feedback_checksum(SerialFeedback *fb) {
    uint16_t calc = fb->start ^ fb->cmd1 ^ fb->cmd2;
    calc ^= fb->speedR_meas ^ fb->speedL_meas;
    calc ^= fb->batVoltage ^ fb->boardTemp;
    calc ^= fb->posR_cm ^ fb->posL_cm;
    calc ^= fb->dcCurrR ^ fb->dcCurrL;
    calc ^= ((uint16_t)fb->errorFlags | ((uint16_t)fb->statusByte << 8));
    return calc == fb->checksum;
}

void create_command(SerialCommand *cmd, int16_t speedL, int16_t speedR) {
    cmd->start = 0xABCD;
    cmd->speedL = speedL;
    cmd->speedR = speedR;
    cmd->checksum = cmd->start ^ cmd->speedL ^ cmd->speedR;
}

// Conversion functions
float feedback_to_speed_kmh(int16_t speed_raw) {
    return (float)speed_raw / 100.0f;
}

float feedback_to_voltage(int16_t voltage_raw) {
    return (float)voltage_raw / 100.0f;
}

float feedback_to_temp(int16_t temp_raw) {
    return (float)temp_raw / 10.0f;
}

float feedback_to_position_m(int16_t pos_raw) {
    return (float)pos_raw / 10.0f;
}

float feedback_to_current_a(int16_t current_raw) {
    return (float)current_raw / 100.0f;
}
```

## Timeout Protection
- If no valid command is received for **~0.8 seconds** (160 × 5ms main loop):
  - Motors are automatically stopped
  - `ERROR_FLAG_TIMEOUT_SERIAL` is set in errorFlags
  - System remains in safe state until valid commands resume

## Notes
- All multi-byte values are transmitted in **little-endian** format (LSB first)
- Frame validation relies on both start frame (0xABCD) and XOR checksum
- Position values are incremental and can overflow (wrap around at ±32767)
- Current values can be negative (regenerative braking)
- Temperature is in °C × 10 (resolution: 0.1°C)
- Battery voltage in V × 100 (resolution: 0.01V)

## Revision History
- **v1.0** (2025-12-11): Initial protocol definition with 28-byte feedback frame including errorFlags and statusByte
