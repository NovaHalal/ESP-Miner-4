# BM1370 UART Quick Reference

## Three Critical Values - Direct Answers

### 1. THE HEARTBEAT (Baud Rate)

**Integer Value:** `115200`

**Location:** `components/asic/include/serial.h:12`
```c
#define UART_FREQ 115200
```

**Baud Rate Negotiation:** **YES** - Two-stage negotiation:
- **Stage 1 (Init):** 115200 baud - for ASIC detection
- **Stage 2 (Operation):** 1000000 baud (1 Mbaud) - for high-speed mining

---

### 2. THE SENTENCE STRUCTURE (C Struct)

```c
// Complete UART Packet Structure
typedef struct __attribute__((__packed__)) {
    // Header Section (Fixed)
    uint8_t  preamble_1;      // Always 0x55
    uint8_t  preamble_2;      // Always 0xAA
    uint8_t  header;          // Type | Group | Command
    uint8_t  length;          // Data length + CRC length
    
    // Data Section (Variable)
    uint8_t  data[N];         // N bytes of data
    
    // Footer Section (Variable)
    union {
        struct {              // For CMD packets
            uint8_t crc5;     // 5-bit CRC in 1 byte
        } cmd;
        struct {              // For JOB packets  
            uint16_t crc16;   // 16-bit CRC in 2 bytes (big-endian)
        } job;
    } footer;
} bm1370_uart_packet_t;
```

**Details:**
- **Header always wraps commands:** YES - `0x55 0xAA` preamble on every packet
- **CRC/Checksum:** YES - CRC5 for commands, CRC16 for jobs
- **CRC Algorithm:** 
  - CRC5: Polynomial x⁵ + x² + 1, init 0x1F
  - CRC16: Polynomial 0x1021, init 0xFFFF

---

### 3. THE "MAGIC" HANDSHAKE

**Answer:** NO traditional magic handshake (no repeated 0x00 bytes)

**Instead:** Structured initialization sequence after reset:

```c
// BM1370_init() sequence (bm1370.c:161-269)

// Step 1: Set version mask (3 times for reliability)
BM1370_set_version_mask(STRATUM_DEFAULT_VERSION_MASK);

// Step 2: Chip detection - THIS IS THE KEY "HANDSHAKE"
// Broadcast read of register 0x00 (Chip ID)
// Packet: 55 AA 52 05 00 00 [CRC5]
_send_BM1370((TYPE_CMD | GROUP_ALL | CMD_READ), 
             (uint8_t[]){0x00, BM_CHIP_ID}, 2, ...);

// Expected response: AA 55 13 70 00 00 00 00 00 00 [CRC]
//                          ^^^^^ = 0x1370 (BM1370 chip ID)

// Step 3: Count responding chips
int chip_counter = count_asic_chips(...);

// Step 4: Configure all chips
// - Write to register 0xA8, 0x18 (Misc Control)
// - Send chain inactive command
// - Assign unique addresses to each chip
// - Configure core registers per-chip
// - Ramp up frequency gradually
```

**Key Point:** The "handshake" is the broadcast Chip ID read command at 115200 baud immediately after hardware reset. No pre-sequence required.

---

## Implementation File Locations

| Component | File | Line(s) |
|-----------|------|---------|
| Baud Rate Definition | `components/asic/include/serial.h` | 12 |
| UART Init | `components/asic/serial.c` | 21-42 |
| Packet Builder | `components/asic/bm1370.c` | 87-120 (`_send_BM1370`) |
| Init Handshake | `components/asic/bm1370.c` | 161-269 (`BM1370_init`) |
| Baud Negotiation | `main/power/asic_init.c` | 40-56 |
| Max Baud Setter | `components/asic/bm1370.c` | 289-297 |
| CRC5 | `components/asic/crc.c` | 6-24 |
| CRC16 | `components/asic/crc.c` | 46-55 |

---

## Complete Example Packets

### Chip ID Read (Handshake)
```
TX: 55 AA 52 05 00 00 [CRC5]
    │  │  │  │  └─┴─ Read register 0x00
    │  │  │  └─ Length: 3 bytes
    │  │  └─ Header: 0x52 (CMD | ALL | READ)
    │  └─ Preamble
    └─ Preamble

RX: AA 55 13 70 00 00 00 00 00 00 [CRC]
         ^^^^^ Chip ID: 0x1370
```

### Misc Control Write
```
TX: 55 AA 51 09 00 18 F0 00 C1 00 [CRC5]
    │  │  │  │  │  │  └──────┴─ Data: F0 00 C1 00
    │  │  │  │  │  └─ Register: 0x18 (Misc Control)
    │  │  │  │  └─ Chip address: 0x00 (broadcast)
    │  │  │  └─ Length: 7 bytes (6 data + 1 CRC)
    │  │  └─ Header: 0x51 (CMD | ALL | WRITE)
    │  └─ Preamble
    └─ Preamble
```

---

*For detailed information, see `UART_COMMUNICATION_PARAMETERS.md`*
