# BM1370 UART Communication Parameters

## Role: Embedded Protocol Engineer
## Target: ESP-Miner-4 codebase (serial.c, asic.c, bm1370.c)

---

## 1. THE HEARTBEAT (Baud Rate)

### Initial Baud Rate
**Value:** `115200` (defined as `UART_FREQ` in `components/asic/include/serial.h:12`)

```c
#define UART_FREQ 115200
```

### Baud Rate Negotiation: YES

The system implements a **two-stage baud rate negotiation**:

1. **Stage 1 - Initialization (Slow):** `115200` baud
   - Set in `SERIAL_init()` (`serial.c:26`)
   - Used for initial ASIC detection and configuration

2. **Stage 2 - High-Speed Operation:** `1000000` baud (1 Mbaud)
   - Configured in `BM1370_set_max_baud()` (`bm1370.c:289-297`)
   - Activated after successful ASIC initialization
   - Controlled by Fast UART Configuration register (0x28)

```c
int BM1370_set_max_baud(void)
{
    // divider of 0 for 3,125,000
    ESP_LOGI(TAG, "Setting max baud of 1000000 ");
    
    unsigned char fast_uart[] = {0x00, FAST_UART_CONFIGURATION, 0x11, 0x30, 0x02, 0x00};
    _send_BM1370((TYPE_CMD | GROUP_ALL | CMD_WRITE), fast_uart, 6, BM1370_SERIALTX_DEBUG);
    return 1000000;
}
```

**Alternative Default Baud Rate:** `115749` baud
- Available via `BM1370_set_default_baud()` (`bm1370.c:281-287`)
- Uses divider of 26 (binary: 11010)
- Formula: `25M/((denominator+1)*8)`

### UART Physical Configuration
- **TX Pin:** GPIO 17
- **RX Pin:** GPIO 18
- **Data Bits:** 8
- **Parity:** None
- **Stop Bits:** 1
- **Flow Control:** None

---

## 2. THE SENTENCE STRUCTURE (Packet Format)

### C Struct Definition

```c
// Command/Job Packet Structure (Transmit)
struct uart_packet {
    uint8_t  preamble[2];     // Always 0x55 0xAA
    uint8_t  header;          // Command/Job type and group targeting
    uint8_t  length;          // Length of data + CRC (3 or 4 bytes)
    uint8_t  data[N];         // Variable length data (N bytes)
    union {
        uint8_t  crc5;        // 5-bit CRC for CMD packets (in 1 byte)
        uint16_t crc16;       // 16-bit CRC for JOB packets (2 bytes)
    };
};
```

### Detailed Packet Structure

#### Header Field (1 byte)
The header byte encodes the packet type and targeting:
```c
#define TYPE_JOB 0x20       // Job packet
#define TYPE_CMD 0x40       // Command packet

#define GROUP_SINGLE 0x00   // Target single chip
#define GROUP_ALL 0x10      // Broadcast to all chips

#define CMD_SETADDRESS 0x00 // Set chip address
#define CMD_WRITE 0x01      // Write register
#define CMD_READ 0x02       // Read register
#define CMD_INACTIVE 0x03   // Set chip inactive
```

Header is composed as: `TYPE | GROUP | CMD`

Examples:
- `0x51` = `TYPE_CMD | GROUP_ALL | CMD_WRITE` (broadcast write)
- `0x41` = `TYPE_CMD | GROUP_SINGLE | CMD_WRITE` (single chip write)
- `0x52` = `TYPE_CMD | GROUP_ALL | CMD_READ` (broadcast read)

#### Length Field (1 byte)
- For **CMD packets:** `data_len + 3` (includes: data + length field + header + CRC5)
- For **JOB packets:** `data_len + 4` (includes: data + length field + header + CRC16)
- Note: The length value represents the total number of bytes from the length field onwards

#### CRC/Checksum
- **CMD Packets:** Use **CRC5** (5-bit CRC, polynomial: x⁵ + x² + 1, MSB-first)
  - Initial value: `0x1F`
  - Applied to: `header + length + data`
  - Stored in 1 byte (lower 5 bits used)

- **JOB Packets:** Use **CRC16** (16-bit CRC, polynomial 0x1021, initial 0xFFFF)
  - Applied to: `header + length + data`
  - Stored as 2 bytes (big-endian)

### Implementation in Code

The `_send_BM1370()` function (`bm1370.c:87-120`) implements the packet structure:

```c
static void _send_BM1370(uint8_t header, const uint8_t * data, uint8_t data_len, bool debug)
{
    packet_type_t packet_type = (header & TYPE_JOB) ? JOB_PACKET : CMD_PACKET;
    const uint8_t total_length = (packet_type == JOB_PACKET) ? (data_len + 6) : (data_len + 5);
    
    uint8_t buf[total_length];
    
    // add the preamble
    buf[0] = 0x55;
    buf[1] = 0xAA;
    
    // add the header field
    buf[2] = header;
    
    // add the length field
    buf[3] = (packet_type == JOB_PACKET) ? (data_len + 4) : (data_len + 3);
    
    // add the data
    memcpy(buf + 4, data, data_len);
    
    // add the correct crc type
    if (packet_type == JOB_PACKET) {
        uint16_t crc16_total = crc16_false(buf + 2, data_len + 2);
        buf[4 + data_len] = (crc16_total >> 8) & 0xFF;
        buf[5 + data_len] = crc16_total & 0xFF;
    } else {
        buf[4 + data_len] = crc5(buf + 2, data_len + 2);
    }
    
    // send serial data
    SERIAL_send(buf, total_length, debug);
}
```

### Example Packets

#### Example 1: Broadcast Read Chip ID
```
55 AA 52 05 00 00 [CRC5]
│  │  │  │  └─┴─ Data: Read register 0x00 (Chip ID)
│  │  │  └─ Length: 3 (2 data bytes + 1 CRC5)
│  │  └─ Header: 0x52 (TYPE_CMD | GROUP_ALL | CMD_READ)
│  └─ Preamble byte 2
└─ Preamble byte 1
```

#### Example 2: Broadcast Write to Misc Control Register
```
55 AA 51 09 00 18 F0 00 C1 00 [CRC5]
│  │  │  │  └──────────────┴─ Data: Write to reg 0x18, value F0 00 C1 00
│  │  │  └─ Length: 7 (6 data bytes + 1 CRC5)
│  │  └─ Header: 0x51 (TYPE_CMD | GROUP_ALL | CMD_WRITE)
│  └─ Preamble byte 2
└─ Preamble byte 1
```

### Response Packet Structure (from ASIC)

```c
typedef struct __attribute__((__packed__))
{
    uint16_t preamble;                // Bytes 0-1: 0xAA55 (note: reversed)
    union {
        bm1370_asic_result_job_t job; // Bytes 2-9: Nonce response
        bm1370_asic_result_cmd_t cmd; // Bytes 2-9: Register read response
    };
    uint8_t crc             : 5;      // Byte 10, bits 0-4: CRC bits
    uint8_t                 : 2;      // Byte 10, bits 5-6: Reserved
    uint8_t is_job_response : 1;      // Byte 10, bit 7: Job/Command flag
} bm1370_asic_result_t;
```

---

## 3. THE "MAGIC" HANDSHAKE

### Initialization Sequence in `BM1370_init()`

The `BM1370_init()` function (`bm1370.c:161-269`) does **NOT** use a traditional "magic" handshake like sending 0x00 twenty times. Instead, it uses a **structured initialization sequence**:

#### Step-by-Step Initialization:

1. **Set Version Mask (3 times)**
   - Repeated 3 times for reliability
   - Command: Write to register 0xA4, subregister 0x90

2. **Read Chip ID (Chip Detection)**
   ```c
   _send_BM1370((TYPE_CMD | GROUP_ALL | CMD_READ), (uint8_t[]){0x00, BM_CHIP_ID}, 2, ...);
   ```
   - Broadcasts read of register 0x00 (Chip ID)
   - Expected response: `AA 55 13 70 00 00 00 00 00 00 [CRC]`
   - Identifies BM1370 chips (ID: 0x1370)
   - Counts number of responding ASICs

3. **Configure Core Registers**
   - Register 0xA8: Core configuration
   - Register 0x18: Misc Control (`0xF0 0x00 0xC1 0x00`)
   
4. **Chain Inactive Command**
   ```c
   _send_BM1370((TYPE_CMD | GROUP_ALL | CMD_INACTIVE), (uint8_t[]){0x00, 0x00}, 2, ...);
   ```
   - Prepares chip address assignment

5. **Address Assignment**
   - Each chip assigned unique address in 256-address space
   - Address interval = `256 / chip_count`
   - Example: 4 chips → addresses 0, 64, 128, 192

6. **Per-Chip Configuration**
   - Individual register configuration for each detected chip
   - Registers: 0xA8, 0x18, 0x3C (multiple writes)

7. **Frequency Ramp-Up**
   ```c
   do_frequency_transition(frequency, BM1370_send_hash_frequency);
   ```
   - Gradually increases hash frequency to target
   - Prevents power surges and instability

8. **Final Configuration**
   - Register 0xB9, 0x54, 0x3C writes
   - Register 0x10: Hash counting configuration

### No Pre-Reset Handshake Required

Unlike some protocols, the BM1370 **does not require**:
- Sending null bytes (0x00) repeatedly
- Wake-up sequences
- Special UART break conditions

The chip responds immediately after hardware reset to properly formatted command packets at 115200 baud.

---

## Summary

### Quick Reference

| Parameter | Value |
|-----------|-------|
| **Initial Baud Rate** | 115200 |
| **Operating Baud Rate** | 1000000 (1 Mbaud) |
| **Preamble** | `0x55 0xAA` (all packets) |
| **CRC for Commands** | CRC5 (1 byte) |
| **CRC for Jobs** | CRC16 (2 bytes) |
| **Chip Detection** | Read register 0x00 (broadcasts) |
| **Expected Chip ID** | 0x1370 |
| **Response Preamble** | `0xAA 0x55` (reversed) |

### Files Reference
- **Baud Rate Definition:** `components/asic/include/serial.h:12`
- **UART Initialization:** `components/asic/serial.c:21-42`
- **Packet Formatting:** `components/asic/bm1370.c:87-120`
- **Initialization Sequence:** `components/asic/bm1370.c:161-269`
- **CRC Implementations:** `components/asic/crc.c`
- **ASIC Init Flow:** `main/power/asic_init.c:11-67`

---

*Document generated by analyzing ESP-Miner-4 source code (BM1370 ASIC driver)*
