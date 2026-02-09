# Hardware Audit Documentation Index

## 📚 Available Documentation

This directory contains comprehensive hardware audit documentation for the ESP-Miner-4 (Bitaxe Gamma 601) codebase.

### 🎯 Start Here: Quick Answers

**File:** [`AUDIT_RESULTS_SUMMARY.md`](AUDIT_RESULTS_SUMMARY.md) (11 KB)

Direct answers to the audit requirements:
- ❌ **SPI CS Pin:** DOES NOT EXIST (system uses UART)
- ✅ **GPIO 10 Conflict:** Currently assigned to ASIC_ENABLE
- ✅ **I2C Pins:** GPIO 47 (SDA), GPIO 48 (SCL)
- ✅ **Complete GPIO Map:** 10 pins documented with functions

### 📖 Detailed Analysis

**File:** [`HARDWARE_AUDIT_REPORT.md`](HARDWARE_AUDIT_REPORT.md) (14 KB)

Complete technical audit including:
- Full hardware layout analysis
- Communication interface details (UART, I2C)
- Power tree verification and TPS546 configuration
- Board version specifications
- Pin conflict resolution recommendations
- Source code references

### 📋 Quick Reference Guide

**File:** [`GPIO_QUICK_REFERENCE.md`](GPIO_QUICK_REFERENCE.md) (4.5 KB)

One-page reference for:
- GPIO pin lookup table
- Communication interface summary
- Power tree diagram
- Modification instructions
- Verification commands

---

## 🔑 Key Findings

### 1. NO SPI Interface
- **Finding:** ESP-Miner-4 does NOT use SPI for ASIC communication
- **Actual Protocol:** UART-based serial (BM1370 proprietary)
- **Pins Used:** GPIO 17 (TX), GPIO 18 (RX)

### 2. GPIO Pin Map

| Pin | Function              | Macro              |
|-----|-----------------------|--------------------|
| 0   | Boot Button           | GPIO_BUTTON_BOOT   |
| 1   | ASIC Reset            | GPIO_ASIC_RESET    |
| 10  | ASIC Power Enable     | GPIO_ASIC_ENABLE   |
| 12  | Plug Sense            | GPIO_PLUG_SENSE    |
| 17  | UART TX (ASIC)        | ECHO_TEST_TXD      |
| 18  | UART RX (ASIC)        | ECHO_TEST_RXD      |
| 39  | BAP UART TX           | GPIO_BAP_TX        |
| 40  | BAP UART RX           | GPIO_BAP_RX        |
| 47  | I2C SDA               | GPIO_I2C_SDA       |
| 48  | I2C SCL               | GPIO_I2C_SCL       |

### 3. I2C Power Tree
- **SDA:** GPIO 47
- **SCL:** GPIO 48
- **TPS546 Address:** 0x24
- **Bus Speed:** 400 kHz

---

## 📁 Document Structure

```
.
├── AUDIT_RESULTS_SUMMARY.md    ← Start here for quick answers
├── HARDWARE_AUDIT_REPORT.md    ← Full technical analysis
├── GPIO_QUICK_REFERENCE.md     ← Quick lookup reference
└── README_AUDIT_DOCS.md        ← This index file
```

---

## 🚀 How to Use These Documents

### For Quick Lookup:
→ Use **GPIO_QUICK_REFERENCE.md**

### For Problem Solving:
→ Use **AUDIT_RESULTS_SUMMARY.md**

### For Deep Technical Analysis:
→ Use **HARDWARE_AUDIT_REPORT.md**

---

## 🔧 Modification Guide

### To Change GPIO Pin Assignments:

1. **Edit Kconfig:**
   ```
   File: main/Kconfig.projbuild
   Line: See specific pin in documentation
   ```

2. **Rebuild:**
   ```bash
   idf.py menuconfig
   idf.py build
   idf.py flash
   ```

3. **Verify:**
   ```bash
   grep -r "CONFIG_GPIO" main/
   ```

---

## 📊 Audit Scope

**Repository:** github.com/NovaHalal/ESP-Miner-4  
**Target Board:** Bitaxe Gamma 601  
**ASIC:** BM1370 (1x chip, 128 cores)  
**Firmware:** ESP-IDF v5.x  
**Date:** February 9, 2026  

**Files Analyzed:** 42+  
**Lines Reviewed:** ~15,000  
**Interfaces Documented:** UART (2), I2C (1)  
**GPIO Pins Mapped:** 10  

---

## ✅ Requirements Fulfilled

- ✅ Searched for SPI CS pin (result: does not exist)
- ✅ Traced spi_device_interface_config_t (result: not used)
- ✅ Mapped complete GPIO layout
- ✅ Identified all GPIO macros and values
- ✅ Verified TPS546 I2C pins
- ✅ Documented power tree
- ✅ Provided modification instructions

---

## 📞 Support

For questions about these findings:
1. Review the appropriate documentation file
2. Check source code references provided
3. Use verification commands to confirm findings

---

**Last Updated:** 2026-02-09  
**Branch:** copilot/find-cs-pin-and-map-hardware-layout  
**Commit:** fda8123
