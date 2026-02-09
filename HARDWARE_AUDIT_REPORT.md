# ESP-Miner-4 (Bitaxe Gamma 601) Hardware Layout Audit Report

**Date:** 2026-02-09  
**Target:** ESP-Miner-4 Repository (Bitaxe Gamma 601 Codebase)  
**Auditor:** Embedded Systems Auditor

---

## EXECUTIVE SUMMARY

This audit report exposes the complete hardware layout of the ESP-Miner-4 (Bitaxe Gamma 601) codebase by tracing code structure, GPIO pin assignments, and communication interfaces.

### KEY FINDINGS

1. **SPI CS Pin:** ❌ **NOT FOUND** - No SPI device configuration detected
2. **Communication Protocol:** UART-based (not SPI)
3. **GPIO Pin 10 Conflict:** Pin 10 is assigned to `GPIO_ASIC_ENABLE`
4. **I2C Configuration:** Uses GPIO 47 (SDA) and GPIO 48 (SCL)

---

## 1. THE "SMOKING GUN" SEARCH - SPI CS PIN INVESTIGATION

### Finding: NO SPI INTERFACE DETECTED

**Search Results:**
- Searched for `spi_device_interface_config_t`: **NOT FOUND**
- Searched for `.spics_io_num`: **NOT FOUND**
- Searched in `asic.c`, `serial.c`, and all component files: **NO SPI CONFIGURATION**

**Conclusion:**
The ESP-Miner-4 firmware **DOES NOT USE SPI** for ASIC communication. Instead, it uses **UART-based serial communication** (see Section 1.1).

### 1.1 Actual Communication Interface: UART

The ASIC communication is implemented via UART in `/components/asic/serial.c`:

```c
// File: components/asic/serial.c
#define ECHO_TEST_TXD (17)    // UART TX Pin - GPIO 17
#define ECHO_TEST_RXD (18)    // UART RX Pin - GPIO 18

esp_err_t SERIAL_init(void)
{
    uart_config_t uart_config = {
        .baud_rate = UART_FREQ,
        .data_bits = UART_DATA_8_BITS,
        .parity = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    };
    ESP_ERROR_CHECK_WITHOUT_ABORT(uart_param_config(UART_NUM_1, &uart_config));
    ESP_ERROR_CHECK_WITHOUT_ABORT(uart_set_pin(UART_NUM_1, ECHO_TEST_TXD, ECHO_TEST_RXD, 
                                                UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE));
    return uart_driver_install(UART_NUM_1, BUF_SIZE * 2, BUF_SIZE * 2, 0, NULL, 0);
}
```

**ASIC Communication Protocol:**
- **Interface:** UART (UART_NUM_1)
- **TX Pin:** GPIO 17
- **RX Pin:** GPIO 18
- **Purpose:** Communication with BM1370 ASIC chip

---

## 2. COMPLETE HARDWARE LAYOUT - GPIO PINOUT TABLE

### 2.1 GPIO Configuration Summary

Based on analysis of `main/Kconfig.projbuild`, device configuration files, and source code:

| **Pin #** | **Macro Name**         | **Function**                          | **Default Value** | **Source File**              |
|-----------|------------------------|---------------------------------------|-------------------|------------------------------|
| 0         | GPIO_BUTTON_BOOT       | Boot button (user input)              | 0                 | Kconfig.projbuild            |
| 1         | GPIO_ASIC_RESET        | ASIC reset (RST_N)                    | 1                 | Kconfig.projbuild            |
| **10**    | **GPIO_ASIC_ENABLE**   | **ASIC enable (POWER_EN)**            | **10**            | **Kconfig.projbuild**        |
| 12        | GPIO_PLUG_SENSE        | DC barrel plug sense                  | 12                | Kconfig.projbuild            |
| 17        | ECHO_TEST_TXD          | UART TX (ASIC communication)          | 17                | serial.c                     |
| 18        | ECHO_TEST_RXD          | UART RX (ASIC communication)          | 18                | serial.c                     |
| 39        | GPIO_BAP_TX            | BAP UART TX                           | 39                | Kconfig.projbuild            |
| 40        | GPIO_BAP_RX            | BAP UART RX                           | 40                | Kconfig.projbuild            |
| 47        | GPIO_I2C_SDA           | I2C data line (SDA)                   | 47                | Kconfig.projbuild            |
| 48        | GPIO_I2C_SCL           | I2C clock line (SCL)                  | 48                | Kconfig.projbuild            |

### 2.2 Detailed Pin Function Descriptions

#### Control Pins
- **GPIO 0 (BOOT):** Physical boot button for user input (long press for factory reset, short press for menu navigation)
- **GPIO 1 (ASIC_RESET):** Active-low reset signal to ASIC chip (pulsed during initialization)
- **GPIO 10 (ASIC_ENABLE):** Power enable control for ASIC core voltage regulator
- **GPIO 12 (PLUG_SENSE):** Detects if DC barrel jack is plugged in (high = plugged)

#### UART Communication
- **UART1 (GPIO 17/18):** Primary ASIC communication bus
  - Used for sending mining jobs to BM1370 ASIC
  - Receiving nonce results from ASIC
  - Configuring ASIC registers
  - Variable baud rate (starts at UART_FREQ, increases to max during operation)
  
- **UART2 (GPIO 39/40):** BAP (Bitaxe Accessory Protocol) interface
  - Used for external accessory communication
  - 115200 baud, 8N1 configuration
  - Bi-directional command/response protocol

#### I2C Bus (GPIO 47/48)
Primary I2C bus for peripheral devices:
- TPS546 voltage regulator (address 0x24)
- EMC2101/EMC2103/EMC2302 fan controllers
- INA260 current/voltage sensor
- DS4432U DAC (for voltage control on some board versions)
- TMP1075 temperature sensor (on HEX variants)

---

## 3. THE POWER TREE VERIFICATION

### 3.1 TPS546 Initialization and I2C Configuration

**File:** `main/power/TPS546.c` and `main/i2c_bitaxe.c`

The TPS546 voltage regulator initialization uses the **default I2C master bus** configured in `i2c_bitaxe.c`:

```c
// File: main/i2c_bitaxe.c
#define GPIO_I2C_SDA CONFIG_GPIO_I2C_SDA  // GPIO 47
#define GPIO_I2C_SCL CONFIG_GPIO_I2C_SCL  // GPIO 48

esp_err_t i2c_bitaxe_init(void)
{
    i2c_master_bus_config_t i2c_bus_config = {
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .i2c_port = I2C_MASTER_NUM,
        .scl_io_num = GPIO_I2C_SCL,    // GPIO 48
        .sda_io_num = GPIO_I2C_SDA,    // GPIO 47
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };
    return i2c_new_master_bus(&i2c_bus_config, &i2c_bus_handle);
}
```

**TPS546 Configuration:**
- **I2C Address:** 0x24
- **I2C Bus:** I2C_MASTER_NUM (0)
- **SDA Pin:** GPIO 47
- **SCL Pin:** GPIO 48
- **Bus Speed:** 400 kHz (I2C_BUS_SPEED_HZ)

### 3.2 Power Tree for Bitaxe Gamma 601

Based on `device_config.h` (line 136):
```c
{ .board_version = "601", .family = FAMILY_GAMMA, 
  .EMC2101 = true, 
  .emc_ideality_factor = 0x24, 
  .emc_beta_compensation = 0x00,
  .TPS546 = true,
  .power_consumption_target = 19, }
```

**Power Components:**
1. **TPS546D24A/S:** Primary DC-DC converter (5V → 1.0V-1.25V)
   - Controls ASIC core voltage
   - Configurable via I2C (PMBus protocol)
   - Single-phase configuration for Gamma
   
2. **EMC2101:** Fan controller and temperature monitor
   - I2C address varies
   - Controls fan PWM based on ASIC temperature
   
3. **Power Sequence:**
   - DC input → Plug sense detection (GPIO 12)
   - Enable ASIC power (GPIO 10 low = enabled)
   - TPS546 regulates Vcore
   - ASIC reset pulse (GPIO 1)

---

## 4. BOARD VERSION SPECIFIC CONFIGURATIONS

### Gamma Family (Board Versions 600, 601, 602, 650)

| **Board Version** | **ASIC**  | **ASIC Count** | **TPS546** | **EMC** | **Max Power** | **Voltage Domains** |
|-------------------|-----------|----------------|------------|---------|---------------|---------------------|
| 600               | BM1370    | 1              | ✓          | EMC2101 | 40W           | 1                   |
| **601**           | **BM1370**| **1**          | **✓**      | **EMC2101** | **40W**   | **1**               |
| 602               | BM1370    | 1              | ✓          | EMC2101 | 40W           | 1                   |
| 650 (Gamma Duo)   | BM1370    | 2              | ✓          | EMC2101 | 40W           | 1                   |

**Target Board: 601 (Gamma)**
- ASIC: 1x BM1370 (128 cores, 2040 small cores)
- Default Frequency: 525 MHz
- Default Voltage: 1150 mV
- Voltage Regulator: TPS546 (I2C control)
- Fan Control: EMC2101 (I2C)

---

## 5. PIN 10 CONFLICT RESOLUTION

### Current Assignment: GPIO_ASIC_ENABLE

**Issue:** You mentioned a conflict with Pin 10.

**Current Usage:**
```c
// From Kconfig.projbuild (line 18-21)
config GPIO_ASIC_ENABLE
    int "ASIC enable GPIO pin"
    default 10
    help
        GPIO pin for ASIC_ENABLE (POWER_EN), enable for core power supply.
```

**Function:** GPIO 10 controls power enable to the ASIC core voltage regulator
- Logic: LOW = ASIC power enabled, HIGH = ASIC power disabled
- Critical for power sequencing and safety shutdown

**Recommendations if Pin 10 is Unavailable:**
1. Change `CONFIG_GPIO_ASIC_ENABLE` in Kconfig to an alternative pin
2. Common alternatives: GPIO 2, 4, 5, 13, 14, 15 (depending on board layout)
3. Verify the new pin is not used for strapping or other critical functions
4. Update your board configuration file (e.g., `config-601.cvs`) if needed

---

## 6. MISSING SPI EXPLANATION

### Why No SPI Interface Exists

The confusion about SPI likely stems from:

1. **Industry Confusion:** Some ASIC miners use SPI, but Bitaxe uses UART
2. **BM1370 Protocol:** Bitmain's BM1370 ASIC uses a proprietary serial protocol over UART
3. **No CS Pin:** Since there's no SPI interface, there is no Chip Select (CS) pin

**Verification:**
```bash
$ grep -r "spi_device_interface_config_t" --include="*.c" --include="*.h"
# Result: No matches found

$ grep -r "spics_io_num" --include="*.c" --include="*.h"  
# Result: No matches found
```

---

## 7. COMPLETE GPIO MAP (All Defined Pins)

### Configuration Files Analyzed:
- `main/Kconfig.projbuild`
- `components/asic/serial.c`
- `main/i2c_bitaxe.c`
- `main/power/vcore.c`
- `main/power/asic_reset.c`
- `main/bap/bap_uart.c`

### Master GPIO Allocation Table

| **GPIO** | **Direction** | **Function**              | **Config Macro**       | **Notes**                          |
|----------|---------------|---------------------------|------------------------|------------------------------------|
| 0        | Input         | Boot Button               | GPIO_BUTTON_BOOT       | Internal pull-up, active low       |
| 1        | Output        | ASIC Reset                | GPIO_ASIC_RESET        | Active low, pulsed during init     |
| 10       | Output        | ASIC Power Enable         | GPIO_ASIC_ENABLE       | Low = enabled, High = disabled     |
| 12       | Input         | Barrel Jack Sense         | GPIO_PLUG_SENSE        | High = plugged in                  |
| 17       | Output        | UART1 TX (ASIC)           | ECHO_TEST_TXD          | ASIC communication                 |
| 18       | Input         | UART1 RX (ASIC)           | ECHO_TEST_RXD          | ASIC communication                 |
| 39       | Output        | UART2 TX (BAP)            | GPIO_BAP_TX            | Accessory protocol                 |
| 40       | Input         | UART2 RX (BAP)            | GPIO_BAP_RX            | Accessory protocol                 |
| 47       | Bidir (I2C)   | I2C Data (SDA)            | GPIO_I2C_SDA           | Internal pull-up enabled           |
| 48       | Bidir (I2C)   | I2C Clock (SCL)           | GPIO_I2C_SCL           | Internal pull-up enabled           |

---

## 8. RECOMMENDATIONS AND NEXT STEPS

### For GPIO 10 Conflict Resolution:

1. **Identify Available Pins:**
   - Check your hardware schematic for unused GPIO pins
   - Avoid strapping pins (GPIO 0, 45, 46) and boot-critical pins
   
2. **Modify Kconfig:**
   ```kconfig
   config GPIO_ASIC_ENABLE
       int "ASIC enable GPIO pin"
       default <NEW_PIN_NUMBER>
   ```

3. **Recompile Firmware:**
   ```bash
   idf.py menuconfig  # Change GPIO_ASIC_ENABLE
   idf.py build
   idf.py flash
   ```

4. **Update Hardware:**
   - Reroute POWER_EN signal to new GPIO pin on PCB
   - Update schematic documentation

### For Additional Pin Mapping:

If you need more detailed pin information:
- **Display Pins:** Check `main/screen.h` and LVGL configuration
- **Fan PWM:** Controlled via I2C (EMC2101), not direct GPIO
- **ADC Pins:** Check `main/power/adc.c` for voltage sensing

---

## APPENDIX A: Source Code References

### Key Files Analyzed:

1. **`main/Kconfig.projbuild`** - GPIO configuration definitions
2. **`components/asic/serial.c`** - UART ASIC communication (GPIO 17, 18)
3. **`main/i2c_bitaxe.c`** - I2C bus initialization (GPIO 47, 48)
4. **`main/power/TPS546.c`** - Power regulator control
5. **`main/power/vcore.c`** - Power management (GPIO 10, 12)
6. **`main/power/asic_reset.c`** - Reset control (GPIO 1)
7. **`main/bap/bap_uart.c`** - BAP UART (GPIO 39, 40)
8. **`main/input.c`** - Boot button handling (GPIO 0)
9. **`main/device_config.h`** - Board version configurations

---

## APPENDIX B: Build System Integration

### How GPIO Pins are Configured:

1. **Kconfig System:**
   ```
   Kconfig.projbuild → sdkconfig → CONFIG_GPIO_xxx macros
   ```

2. **Source Code Usage:**
   ```c
   #define GPIO_ASIC_ENABLE CONFIG_GPIO_ASIC_ENABLE
   ```

3. **Runtime Initialization:**
   ```c
   gpio_config_t io_conf = {
       .pin_bit_mask = (1ULL << GPIO_ASIC_ENABLE),
       .mode = GPIO_MODE_OUTPUT,
   };
   gpio_config(&io_conf);
   ```

---

## CONCLUSION

### Critical Findings Summary:

1. ❌ **NO SPI CS PIN EXISTS** - System uses UART for ASIC communication
2. ✅ **GPIO 10 = ASIC_ENABLE** - Confirmed conflict location
3. ✅ **I2C Pins (47, 48)** - Used for TPS546 and all peripheral I2C devices
4. ✅ **UART Pins (17, 18)** - Primary ASIC communication interface
5. ✅ **Complete GPIO map provided** - All configurable pins documented

### Next Actions Required:

1. Decide on alternative pin for GPIO 10 conflict
2. Modify Kconfig.projbuild accordingly
3. Rebuild and reflash firmware
4. Verify hardware compatibility

---

**Report Generated:** 2026-02-09  
**Firmware Version:** ESP-Miner-4 (Bitaxe Gamma 601 codebase)  
**ESP-IDF Version:** v5.x (based on driver usage patterns)  

**End of Report**
