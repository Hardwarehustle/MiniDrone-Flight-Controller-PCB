# 🚁 MiniDrone Flight Controller PCB
## 📌 Project Overview

This is a fully integrated **mini quadcopter flight controller PCB** shaped as a drone frame — the PCB itself acts as the structural frame with four motor mount arms extending from the central compute hub. It packs an **ESP32-S3** SoC (dual-core 240 MHz Xtensa LX7, Wi-Fi + BLE 5.0), an **OV2640 camera** (24-pin FPC), a **MPU-6050 IMU** (6-axis gyro + accelerometer), **LiPo battery management** (TP4056 + DW01A + FS8205A), **4× SI2302A N-MOSFET brushed motor drives**, a **microSD card** slot for SD_MMC, and **USB-C OTG** — all on a single X-shaped blue PCB.

The board is designed for a compact indoor FPV or autonomous mini-drone platform with onboard vision, motion sensing, Wi-Fi/BLE telemetry, and local SD logging.

---

## 🖼️ Project Visuals

### PCB Layout (Top)
![PCB Layout](images/PCB_Layout.jpg)

### Bottom View
![Bottom View](images/BottomView.jpg)

### Schematic
![Schematic](images/SCH.pdf)

---

## 🏗️ System Architecture

```
LiPo Battery ──► MK-12C02 Slide Switch (SW2)
                          │
               TP4056 (U1) Li-ion Charger ◄── USB-C VBUS (charging)
                          │
               DW01A (U2) Battery Protection (OC, OD, TD)
                          │
               FS8205A (Q1) Dual P-MOSFET Protection Switch
                          │
                       Vout Rail
                          │
          ┌───────────────┼───────────────────────────┐
          ▼               ▼                           ▼
  XC6220B331 LDO    ME6211C28 LDO (2.8V)   ME6211C15 LDO (1.5V)
  (+3.3V for SoC)   (OV2640 DVDD/AVDD)     (OV2640 core)
          │
       +3.3VC Rail
          │
   ┌──────┼──────────────────────────────────────┐
   ▼      ▼              ▼           ▼           ▼
ESP32-S3  MPU-6050     SD_MMC     USB-C OTG   SI2302A×4
(SoC)   (Gyro/Accel)  (CARD1)   (TYPE-C16P)  (Motors H1-H4)
   │
   └──► OV2640 Camera (24P FPC, U5)
```

---

## 📊 Component BOM

### Core ICs

| Ref | Part | Function |
|:---:|:---|:---|
| ESP32-S3-WROOM-1-N16R2 | ESP32-S3 SoC | Dual-core 240 MHz, Wi-Fi + BLE 5.0, 16MB Flash, 2MB PSRAM |
| U1 | TP4056-42-ESOP8 | 1A Li-ion/LiPo battery charger |
| U2 | DW01A | Battery protection IC (OC / OD / short circuit) |
| Q1 | FS8205A | Dual N-MOSFET battery protection switch |
| U3 | XC6220B331PR-G | 300mA LDO regulator — Vout → +3.3V |
| ME6211C28M5G-N1 | ME6211C28 | 300mA LDO — 2.8V for OV2640 |
| ME6211C15M5G-N2 | ME6211C15 | 300mA LDO — 1.5V for OV2640 core |
| MPU-1 | MPU-6050 | 6-axis IMU (3-axis gyro + 3-axis accel), I2C |
| U5 | ZX-0.5FPC-FWX-H1.5-24P | OV2640 camera 24-pin FPC connector |
| CARD1 | TF PUSH | MicroSD card (SD_MMC 4-bit interface) |
| USB1 | TYPE-C 16PIN | USB-C OTG connector |
| D2 | USBLC6-2SC6 | USB ESD protection TVS |

### Motor Drive MOSFETs

| Ref | Part | GPIO | Motor |
|:---:|:---:|:---:|:---:|
| Q2 | SI2302A-TP | GPIO2 | Motor H1 (Front-Left) |
| Q3 | SI2302A-TP | GPIO1 | Motor H2 (Front-Right) |
| Q4 | SI2302A-TP | GPIO3 | Motor H3 (Rear-Left) |
| Q5 | SI2302A-TP | GPIO39 | Motor H4 (Rear-Right) |

Each SI2302A N-MOSFET gate is driven directly from ESP32-S3 PWM GPIO through a 10kΩ gate resistor, and is protected by an SS12 Schottky flyback diode (D5–D8).

---

## 🔌 ESP32-S3 GPIO Assignment

| GPIO | Signal | Block |
|:---:|:---|:---|
| GPIO1 | Motor H2 (PWM) | MOSFET Q3 |
| GPIO2 | Motor H1 (PWM) | MOSFET Q2 |
| GPIO3 | Motor H3 (PWM) | MOSFET Q4 |
| GPIO39 | Motor H4 (PWM) | MOSFET Q5 |
| GPIO4 | CAM_XCLK | OV2640 Clock |
| GPIO5 | CAM_VSYNC | OV2640 |
| GPIO6 | CAM_RESET | OV2640 |
| GPIO7 | CAM_PWDN | OV2640 Power Down |
| GPIO8–GPIO18 | CAM_D0–D7, PCLK, HREF, SIOC, SIOD | OV2640 data + I2C |
| GPIO19 | USB D+ | USB OTG |
| GPIO20 | USB D− | USB OTG |
| GPIO21 | Status LED | Green-yellow LED |
| GPIO35 | SD_CMD | SD_MMC |
| GPIO36 | SD_CLK | SD_MMC |
| GPIO37 | SD_DAT0 | SD_MMC |
| GPIO47 | MPU-6050 SCL | I2C |
| GPIO48 | MPU-6050 SDA | I2C |
| GPIO0 | BOOT | Boot mode select |
| EN | RESET | Hardware reset |

---

## 🔋 Battery Management System

The BMS consists of three stages working together to safely charge and discharge the LiPo cell:

### Stage 1 — Charging: TP4056
- Max charge current: **1A** (set by PROG pin resistor R2 = 1.2kΩ)
- Charge voltage: **4.2V** (fixed internal reference)
- Charging via USB-C VBUS
- CHRG# pin drives **LED_Red** (charging indicator)
- STDBY# pin drives **LED_Green** (charge complete)

### Stage 2 — Protection: DW01A
- Overcurrent protection (OC)
- Over-discharge protection (OD): cuts off battery below ~2.4V
- Short circuit protection (TD)
- Operates with external dual MOSFET (FS8205A)

### Stage 3 — Switching: FS8205A
- Dual N-channel MOSFET in a single SOT-23-6 package
- G1 controlled by OD pin of DW01A
- G2 controlled by OC pin of DW01A
- Both protect battery from damage during abnormal conditions

---

## 📷 OV2640 Camera Interface

The OV2640 is a 2MP image sensor from OmniVision, connected via a **24-pin 0.5mm pitch FPC connector** (ZX-0.5FPC-FWX-H1.5-24P, U5). It requires three regulated power supplies generated by dedicated ME6211 LDOs:

| Rail | Voltage | LDO | Purpose |
|:---:|:---:|:---:|:---|
| AVDD / DOVDD | 2.8V | ME6211C28M5G-N1 | Analog + I/O power |
| DVDD | 1.5V | ME6211C15M5G-N2 | Digital core power |

Camera control pins from ESP32-S3:
- **CAM_RESET** (GPIO6) — hardware reset
- **CAM_PWDN** (GPIO7) — power down mode
- **SIOC/SIOD** — I2C configuration bus (SCCB protocol)
- **GPIO4–GPIO18** — parallel DVP data bus (D0–D7, PCLK, VSYNC, HREF)

---

## 🎯 MPU-6050 IMU (Gyro + Accel)

The **MPU-6050** is a 6-axis MEMS IMU providing 3-axis gyroscope and 3-axis accelerometer data over I2C at up to 400 kHz. It is connected to:
- **SDA → GPIO48**, **SCL → GPIO47**
- I2C pull-ups R12 and R13 (10kΩ each) to +3.3VC
- INT pin available for interrupt-driven data-ready notification
- FSYNC pin available for frame sync with camera
- Decoupling: C16 (2.2nF) + C17 (100nF) on VDD

The MPU-6050 data feeds the flight stabilization loop running on the ESP32-S3 for attitude control (roll, pitch, yaw).

---

## 💾 MicroSD Card (SD_MMC)

The microSD card (CARD1 — TF PUSH type) operates in **SD_MMC 4-bit wide bus mode** for high-speed data logging:

| Signal | GPIO | Description |
|:---:|:---:|:---|
| SD_CMD | GPIO35 | Command line |
| SD_CLK | GPIO36 | Clock |
| SD_DAT0 | GPIO37 | Data bit 0 |

Pull-up resistors R14–R17 (10kΩ) hold all SD_MMC lines high when idle.

---

## 🔌 USB-C OTG

The **TYPE-C 16-pin connector** (USB1) supports:
- **USB 2.0 Full-Speed** native on ESP32-S3 (GPIO19/GPIO20 as D+/D−, 90Ω differential pair)
- **OTG** capability — can act as USB device or host
- CC1/CC2 lines with R6/R7 (5.1kΩ) for USB-C cable orientation and current advertisement
- **USBLC6-2SC6 TVS array (D2)** for full ESD protection on DP/DM lines
- VBUS routing for charging input to TP4056

---

## 🔄 Auto-Reset and Boot Circuit

| Signal | GPIO | Component | Function |
|:---:|:---:|:---:|:---|
| RST | EN | C20 (100nF) | Reset filter capacitor |
| BOOT | GPIO0 | C21 (100nF) | Boot mode filter cap |
| RST | — | RESET button | Manual reset |
| BOOT | — | BOOT button | Manual boot mode entry |
| GPIO21 | — | Green-yellow LED (KT-0805YG) + R21 1kΩ | Status indicator |

---

## ⚡ Power Architecture Summary

```
USB-C VBUS (5V) ──► TP4056 charger ──► LiPo Battery (3.7V)
                                              │
                                        DW01A + FS8205A
                                              │
                                           Vout
                          ┌───────────────────┼────────────────────┐
                          ▼                   ▼                    ▼
                  XC6220B331 LDO       ME6211C28 (2.8V)    ME6211C15 (1.5V)
                   → +3.3V              → OV2640 AVDD         → OV2640 DVDD
                   ESP32-S3
                   MPU-6050
                   SD Card
                          │
                  4× SI2302A → Brushed Motors (direct Vout)
```

---

## 📐 PCB Design Highlights

- **Tool:** EasyEDA V1.0
- **Board shape:** X-frame quadcopter — PCB is simultaneously the structural drone frame
- **Arm ends:** 4× motor mount clamps (H1–H4) at arm tips
- **Center hub:** ESP32-S3, camera FPC, MPU-6050, SD card, USB-C all centrally placed
- **Color:** Blue PCB with gold pads (ENIG finish recommended)
- **Layers:** 2-layer
- **USB differential pair:** Routed as 90Ω controlled impedance

---

## 🧪 Bring-Up Procedure

### Step 1 — Power Check
- Connect LiPo battery and slide SW2 to ON
- Measure Vout (3.5–4.2V), +3.3V at ESP32 VDD, 2.8V at OV2640, 1.5V at OV2640 DVDD

### Step 2 — USB and Programming
- Connect USB-C to PC
- Press BOOT + RESET (or use auto-reset via DTR/RTS if host-side bridge added)
- Flash firmware using ESP-IDF or Arduino IDE (ESP32-S3 target)

### Step 3 — IMU Test
- I2C scan on GPIO47/GPIO48 — expect MPU-6050 at address 0x68
- Read accel and gyro registers — verify non-zero output when board is tilted

### Step 4 — Camera Test
- Enable OV2640 power (ME6211 CE pins asserted)
- Init SCCB (I2C) to configure OV2640 registers
- Read a JPEG frame — verify image over Wi-Fi or SD

### Step 5 — SD Card Test
- Mount FAT32 SD card
- Write a test log file — verify read-back

### Step 6 — Motor Test
- Apply PWM to GPIO1–GPIO3 and GPIO39 at low duty (5–10%)
- Verify each SI2302A gate toggles correctly with oscilloscope
- Verify motor spins at low throttle safely

---

## ⚠️ Design Notes & Cautions

- **SI2302A gate voltage** — ESP32-S3 GPIO drives 3.3V; SI2302A Vgs(th) ≈ 0.9V min, ensure full enhancement at 3.3V drive
- **SS12 flyback diodes (D5–D8)** — essential for brushed motor inductive kickback protection; place close to MOSFET drain
- **OV2640 power sequence** — apply 2.8V before 1.5V per OV2640 datasheet power-up sequence
- **90Ω USB differential pair** — GPIO19/GPIO20 traces must be routed as a matched-impedance differential pair; keep equal length and spacing
- **MPU-6050 placement** — mount at the board center of mass for accurate attitude measurement; avoid placing near switching regulators
- **TP4056 PROG resistor (R2 = 1.2kΩ)** — sets charge current to ~1A; verify LiPo cell can handle this charge rate (1C minimum rated capacity)
- **PCB-as-frame structural note** — board is structural; use FR4 1.6mm minimum thickness; motor clamp holes should be reinforced with copper rings

---

## 📄 License

This project is open for educational and personal use.  
© 2026 Janardhan BV — All rights reserved.

---

## 🙋 Author

**Janardhan BV**  
Embedded Hardware Engineer | PCB Design | Robotics  
📍 Bengaluru, India

---
*MiniDrone — ESP32-S3 Flight Controller PCB | Designed in EasyEDA*
