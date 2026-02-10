# Bitaxe Gamma 601 Architecture Analysis

## Overview

A comprehensive architectural analysis of the Bitaxe Gamma 601 (ESP-Miner-4) has been completed and documented using the NovaMaths Architecture framework. This analysis provides deep insights into the hardware layout, inter-component communication, logical flow, and safety mechanisms.

## Document Location

📄 **Main Document:** `/doc/BITAXE_GAMMA_601_ARCHITECTURE_ANALYSIS.md`

## Key Sections Covered

### 1. Hardware Architecture & Component Map
- Complete hardware component inventory with I2C addresses
- BM1370 ASIC specifications (1.07 TH/s @ 525 MHz)
- GPIO pin assignments and power domains
- Communication protocol specifications

### 2. Inter-Component Communication Protocols
- **I2C Bus:** 400 kHz multi-drop configuration
  - TPS546 (0x24) - Voltage regulator (PMBus)
  - DS4432U (0x48) - DAC voltage adjustment
  - INA260 (0x40) - Power monitoring
  - EMC2101 (0x4C) - Thermal + fan control
- **UART:** ESP32 ↔ BM1370 ASIC communication (115200 → max baud)
- **SPI:** ESP32 ↔ PSRAM (8MB Octal SPI)

### 3. Logical Flow & Control Sequences
- Complete system initialization sequence (12 steps)
- Mining operation loop (pool ↔ ASIC data flow)
- Voltage control sequence via DS4432U
- PID-based thermal control loop (100ms sampling)

### 4. NovaMaths Framework Analysis

This section maps the Bitaxe system to three core mathematical frameworks:

#### Definition 1: Deformation Field ($\Theta_\nabla$)
- **Concept:** Accumulated state changes and control output variations
- **Mapping:** Temperature gradients, power deltas, frequency transitions
- **Thresholds:** 75°C ASIC, 105°C VR trigger regulation

**Code Location:** `/main/tasks/power_management_task.c` (lines 78-92)

#### Definition 2: Self-Regulation ($\Xi_\lambda$)
- **Concept:** Automatic system behavior constraint when deformation exceeds limits
- **Mapping:** Overheat detection → voltage shutdown → cooling cycle → reduced params recovery
- **Damping Factor:** α ≈ 0.9 (voltage/frequency reduced by 100mV/100MHz)

**Code Location:** `/main/tasks/power_management_task.c` (lines 92-165)

#### Definition 3: Reversion Mechanism ($\Psi^\circlearrowleft$)
- **Concept:** Reset and recovery procedures for sustained regulation or failure
- **Mapping:**
  - **Soft Reset:** GPIO ASIC reset (200ms, preserve ESP32 state)
  - **Recovery Mode:** Post-regulation re-initialization
  - **Hard Reset:** Full ESP32 reboot (esp_restart)

**Code Locations:**
- `/main/power/asic_reset.c` - GPIO reset control
- `/main/power/asic_init.c` - Recovery initialization
- `/main/http_server/http_server_main.c` - System restart API

### 5. Critical Data Paths & Signal Handling
- Mining job flow: Pool → ESP32 → BM1370 → ESP32 → Pool
- Thermal regulation: EMC2101 → PID controller → Fan PWM
- Power monitoring: INA260 → Display/API

### 6. Safety Mechanisms & Thresholds

| Parameter | Throttle | Shutdown | Hysteresis |
|-----------|----------|----------|------------|
| ASIC Temp | 75°C | 90°C | 30°C (safe=45°C) |
| VR Temp | 105°C | 145°C | 10°C |
| Input Voltage | 3.5V | 6.5V | N/A |
| Core Voltage | 1000mV | 1250mV | Clamp |

### 7. System State Machine
```
POWER_ON → INIT_ASIC → MINING_IDLE → MINING_ACTIVE
                ↑                            ↓
                └─── RECOVERY ←─ REGULATION ←┘
                          ↓
                       FAULT
```

### 8. Code Architecture Map
- Directory structure with NovaMaths framework mappings
- Key function call graphs
- Framework element to code location mapping table

## Quick Reference: NovaMaths Mapping

| Framework Element | Code Location | Key Functions |
|------------------|---------------|---------------|
| **Θ∇ Monitoring** | `power_management_task.c` | Lines 78-87 |
| **Θ∇ Thresholds** | `power_management_task.c` | Lines 24-34 |
| **ξλ Trigger** | `power_management_task.c` | Lines 92-110 |
| **ξλ Damping** | `power_management_task.c` | Lines 143-147 |
| **ξλ Cooling Loop** | `power_management_task.c` | Lines 118-141 |
| **Ψ↻ ASIC Reset** | `asic_reset.c` | `asic_reset()` |
| **Ψ↻ Recovery Init** | `asic_init.c` | `asic_initialize(RECOVERY)` |

## Key Findings

### Hardware
- **Single BM1370 ASIC:** 4 hash domains, 2168 total cores
- **TPS546 + DS4432U:** Dual-stage voltage control (coarse + fine adjustment)
- **EMC2101:** Combined thermal sensing and PWM fan control
- **INA260:** High-accuracy power monitoring (1.25mV/mA resolution)

### Control Systems
- **PID Thermal Control:** Kp=3.0, Ki=0.1, Kd=1.0, REVERSE mode, 100ms sample time
- **Power Management:** 1800ms polling, multi-threshold safety (ASIC 75°C, VR 105°C)
- **Overheat Recovery:** Minimum 30s cooling, automatic parameter reduction

### Safety Architecture
- **Multi-level protection:** Temperature, voltage, current, power limits
- **Graceful degradation:** Automatic frequency/voltage reduction on overheat
- **Watchdog timers:** Task monitoring with automatic ESP32 reboot on hang

## Usage

This analysis serves multiple purposes:

1. **System Understanding:** Comprehensive view of hardware and software integration
2. **Debugging Reference:** Detailed mapping of data paths and control sequences
3. **Modification Guide:** Identifies critical code sections for safe customization
4. **Framework Application:** Demonstrates NovaMaths concepts in embedded systems

## Analysis Methodology

The analysis was conducted through:
1. Source code examination of all critical files
2. Hardware configuration analysis (config-601.cvs)
3. Communication protocol tracing (I2C, UART, PMBus)
4. Control loop identification and mathematical modeling
5. Safety mechanism documentation with threshold mapping
6. State machine construction from code logic

## Related Files

- **Hardware Config:** `config-601.cvs`
- **Main Entry:** `main/main.c`
- **Global State:** `main/global_state.h`
- **ASIC Driver:** `components/asic/bm1370.c`
- **Power Management:** `main/tasks/power_management_task.c`
- **Thermal Control:** `main/tasks/fan_controller_task.c`
- **PID Controller:** `main/thermal/PID.c`

## Document Statistics

- **Total Lines:** 592
- **File Size:** 23 KB
- **Sections:** 8 major + appendices
- **Code References:** 50+ direct file/line citations
- **Tables:** 15+ reference tables
- **Diagrams:** State machines, timing diagrams, data flow charts

## Version History

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2026-02-10 | Initial comprehensive analysis with NovaMaths framework integration |

---

**For detailed information, refer to the complete analysis document:**  
📄 `/doc/BITAXE_GAMMA_601_ARCHITECTURE_ANALYSIS.md`
