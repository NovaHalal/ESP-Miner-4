# GPIO Quick Reference - Bitaxe Gamma 601

## CRITICAL ANSWER TO YOUR QUESTIONS

### 1. SPI CS Pin Value: **DOES NOT EXIST**
- ❌ No SPI interface found in codebase
- ❌ No `spi_device_interface_config_t` struct found
- ❌ No `.spics_io_num` assignment found
- ✅ System uses **UART** for ASIC communication (GPIO 17/18)

### 2. GPIO Pin 10 Conflict
- **Current Assignment:** `GPIO_ASIC_ENABLE` (ASIC power enable)
- **Function:** Controls power supply to ASIC core
- **Logic:** LOW = enabled, HIGH = disabled
- **Source:** `main/Kconfig.projbuild` line 19

### 3. I2C Pins for TPS546
- **SDA:** GPIO 47 (`CONFIG_GPIO_I2C_SDA`)
- **SCL:** GPIO 48 (`CONFIG_GPIO_I2C_SCL`)
- **Source:** `main/i2c_bitaxe.c` lines 9-10

---

## COMPLETE GPIO MAP TABLE

| Pin | Macro              | Function              | Default | Source          |
|-----|--------------------|-----------------------|---------|-----------------|
| 0   | GPIO_BUTTON_BOOT   | Boot button           | 0       | Kconfig         |
| 1   | GPIO_ASIC_RESET    | ASIC reset            | 1       | Kconfig         |
| 10  | GPIO_ASIC_ENABLE   | ASIC power enable     | 10      | Kconfig         |
| 12  | GPIO_PLUG_SENSE    | DC plug sense         | 12      | Kconfig         |
| 17  | ECHO_TEST_TXD      | UART1 TX (ASIC)       | 17      | serial.c        |
| 18  | ECHO_TEST_RXD      | UART1 RX (ASIC)       | 18      | serial.c        |
| 39  | GPIO_BAP_TX        | UART2 TX (BAP)        | 39      | Kconfig         |
| 40  | GPIO_BAP_RX        | UART2 RX (BAP)        | 40      | Kconfig         |
| 47  | GPIO_I2C_SDA       | I2C data              | 47      | Kconfig         |
| 48  | GPIO_I2C_SCL       | I2C clock             | 48      | Kconfig         |

---

## COMMUNICATION INTERFACES

### UART1 - ASIC Communication
- **TX:** GPIO 17
- **RX:** GPIO 18
- **Protocol:** BM1370 proprietary serial
- **Usage:** Mining job submission, nonce results

### UART2 - BAP Protocol
- **TX:** GPIO 39
- **RX:** GPIO 40
- **Baud:** 115200
- **Usage:** Bitaxe Accessory Protocol

### I2C - Peripherals
- **SDA:** GPIO 47
- **SCL:** GPIO 48
- **Speed:** 400 kHz
- **Devices:**
  - TPS546 (0x24) - Voltage regulator
  - EMC2101 - Fan controller
  - INA260 - Power monitor
  - DS4432U - DAC (voltage control)

---

## POWER TREE

```
DC Input (12V/5V)
    ↓
GPIO 12 (Plug Sense) ← Detect power
    ↓
GPIO 10 (ASIC Enable) ← Control power
    ↓
TPS546 (I2C @ 0x24, GPIO 47/48) ← Regulate voltage
    ↓
ASIC Core Voltage (1.0V-1.25V)
    ↓
GPIO 1 (Reset) ← Initialize ASIC
    ↓
UART1 (GPIO 17/18) ← Communicate with ASIC
```

---

## PIN 10 CONFLICT RESOLUTION

**To change GPIO_ASIC_ENABLE from pin 10:**

1. Edit `main/Kconfig.projbuild` line 19:
   ```kconfig
   config GPIO_ASIC_ENABLE
       int "ASIC enable GPIO pin"
       default <NEW_PIN>  # Change from 10
   ```

2. Rebuild:
   ```bash
   idf.py menuconfig  # Verify change
   idf.py build
   idf.py flash
   ```

3. **Recommended alternative pins:** 2, 4, 5, 13, 14, 15
   (Verify against your hardware schematic)

---

## FILES TO MODIFY FOR GPIO CHANGES

| File                    | Contents                          |
|-------------------------|-----------------------------------|
| `main/Kconfig.projbuild`| All configurable GPIO defaults    |
| `main/i2c_bitaxe.c`     | I2C pin definitions               |
| `components/asic/serial.c` | UART ASIC communication        |
| `main/bap/bap_uart.c`   | BAP UART pins                     |
| `main/power/vcore.c`    | Power control GPIOs               |
| `main/power/asic_reset.c` | Reset control GPIO              |
| `main/input.c`          | Boot button GPIO                  |

---

## VERIFICATION COMMANDS

```bash
# Search for SPI configuration (should return nothing)
grep -r "spi_device_interface_config_t" --include="*.c" --include="*.h"
grep -r "spics_io_num" --include="*.c" --include="*.h"

# Find all GPIO definitions
grep -r "CONFIG_GPIO" main/ --include="*.c" --include="*.h"

# Check I2C pins
grep -r "GPIO_I2C" main/ --include="*.c" --include="*.h"

# View Kconfig GPIO settings
cat main/Kconfig.projbuild | grep -A 4 "config GPIO"
```

---

## BOARD VERSION: 601 (Gamma)

- **ASIC:** 1x BM1370
- **Cores:** 128 cores, 2040 small cores
- **Frequency:** 525 MHz (default)
- **Voltage:** 1150 mV (default)
- **Power:** 40W max
- **Voltage Regulator:** TPS546 (I2C control)
- **Fan Control:** EMC2101 (I2C)
- **Protocol:** UART-based (not SPI)

---

**For complete details, see:** `HARDWARE_AUDIT_REPORT.md`

**Last Updated:** 2026-02-09
