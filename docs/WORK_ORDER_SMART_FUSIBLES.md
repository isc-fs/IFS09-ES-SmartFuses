# WORK ORDER: SMART FUSIBLES (IFS09)
## Low-Voltage Power Distribution, Multi-Rail DC-DC Regulation & CAN Telemetry Module
**Season:** 2026/2027  
**Department:** Low Voltage & Electronics  
**Author:** Andrés Sánchez de Ágreda  
**Date:** 7 September 2026  
**Revision:** Rev 1.0 (`WO-IFS09-SFUS-001`)  
**Base Template:** `IFS_09_W_Template_Base_CB.docx`  

---

## 1. Executive Summary & Functional Description

### 1.1 Purpose
This Work Order defines the hardware engineering requirements, electrical specifications, component selections, power conversion architecture, and verification testing procedures for designing and fabricating the IFS09 **"Smart Fusibles"** printed circuit board (PCB).

The board replaces and significantly expands upon the previous passive revision (`PCB_FUSIBLES_XT60` v2 - 06/26). While the previous board provided basic fuse splitting with friction-fit XT-60 connectors, it lacked diagnostic feedback, real-time power telemetry, and integrated DC-DC regulation.

The Smart Fusibles board acts as the central low-voltage (LV) power distribution, multi-rail regulation, overcurrent protection, and digital power telemetry hub for the Formula Student vehicle, broadcasting voltage, current, and fuse health over CAN to the vehicle ECU.

### 1.2 System Overview & Module Integration
- **12 Protected Fused Power Channels:** Incorporates 12 Keystone 3568 automotive mini blade fuse circuits with dual-tier blown-fuse detection: an instant hardware visual indicator LED per channel, complemented by real-time software detection via downstream INA bus voltage sensing.
- **Locking Automotive Power Input:** Eliminates non-locking RC-style XT60 connectors in favor of a positive click-latch Molex Mega-Fit 4-circuit vertical header rated up to 23A per contact (46A aggregate continuous capacity on dual 24V/GND circuits).
- **Current & Voltage Sensing on Every Rail:** Features 12 Texas Instruments INA260AIPWR monitors with integrated 2 mΩ precision shunts. All 12 monitors reside on a single shared 400 kHz I2C bus via unique pin-strapped hardware addressing (`0x40` to `0x4B`), eliminating external shunts and I2C multiplexers.
- **Integrated Multi-Rail DC-DC Power Conversion:** Direct step-down conversion from the 24V raw accumulator bus to supply: onboard 3.3V logic (Murata OKI-78SR-3.3), DTI Inverter & motor duo (12V @ 2A), Driverless Optical LiDAR (12V @ 10W isolated), and LattePanda Mission Computer (19V @ 90W high-efficiency synchronous buck).
- **Microcontroller & CAN Telemetry:** An STM32L431CBT6 ARM Cortex-M4 microcontroller samples all 12 INA260 channels via DMA and broadcasts diagnostic telemetry over vehicle CAN via a TI TCAN1042V-Q1 automotive transceiver using conflict-free CAN IDs `0x380`–`0x383`.
- **Automotive Sealed Output Interface:** Routes all protected vehicle branches through a sealed TE Connectivity AMPSEAL connector, building directly upon last year's harness layout for seamless vehicle integration.
- **Power & Ground Plane Pour Architecture:** The LV power input from the Molex Mega-Fit connector pours directly into a full-board continuous ground (GND) return plane on the bottom layer (with ground fills on top), while the +24V supply pours into a dedicated high-current supply plane that makes continuous low-resistance contact with the input terminals of all 12 Keystone fuseholders.
- **2-Layer PCB Stackup:** Cost-effective, robust 2-layer FR4 PCB with heavy 2 oz (70 µm) finished copper on top and bottom layers, satisfying IPC-2152 current-carrying requirements.

---

## 2. System Architecture Diagram

The block diagram below illustrates the power flow, DC-DC conversion hierarchy, INA260 monitoring topology, and CAN communication bus.

```
+-----------------------------------------------------------------------------------------+
|                 24V ACCUMULATOR INPUT (Molex Mega-Fit 4-Pin Header, Positive Latch)      |
|                     [ Pin 1,2: +24V_IN (46A max)  |  Pin 3,4: POWER_GND ]                |
+--------------------------------------------+--------------------------------------------+
                                             |
                                             v
                     +------------------------------------------------+
                     | TVS Clamping (SMCJ33A) & Reverse Polarity Prot |
                     +-----------------------+------------------------+
                                             |
        +------------------------------------+------------------------------------+
        |                                                                         |
        v [Direct 24V Fused Bus]                                                  v [DC-DC Power Conversion]
  +-----------+-----------+-----------+                                     +-----------+-----------+-----------+
  | CH 04-07  | CH 08-10  | CH 11-12  |                                     | 3.3V Logic| 12V 2A DTI| 12V 10W   | 19V 90W   |
  | ECU, DASH | REFRI, ACU| AUX, SPARE|                                     | OKI-78SR  | K7812/LMR | OKI-78SR  | LM5145    |
  | SDC, TSAL | DV ACTUAT |           |                                     | 1.5A Max  | 24W Cont. | LiDAR Isol| LattePanda|
  +-----+-----+-----+-----+-----+-----+                                     +-----+-----+-----+-----+-----+-----+-----+---+\
        |           |           |                                                 |           |           |           |
        |           |           |                                                 v           |           |           |
        |           |           |                                            [STM32L431 &]    v           v           v
        |           |           |                                            [TCAN1042V  ]  CH 01       CH 02       CH 03
        |           |           |                                                           (DTI Logic) (LiDAR)   (LattePanda)
        |           |           |                                                             |           |           |
        v           v           v                                                             v           v           v
+-------+-----------+-----------+-------------------------------------------------------------+-----------+-----------+---+
|                    12x AUTOMOTIVE MINI BLADE FUSES (Keystone 3568 Sockets + Blown Fuse LEDs)                    |
+-------+-----------+-----------+-------------------------------------------------------------+-----------+-----------+---+
        |           |           |                                                             |           |           |
        v           v           v                                                             v           v           v
+-------+-----------+-----------+-------------------------------------------------------------+-----------+-----------+---+
|                   12x TI INA260AIPWR DIGITAL POWER SENSORS (Single Shared I2C Bus @ 400 kHz)                    |
|                              Addresses 0x40 to 0x4B (Pin-Strapped via A0 / A1)                                  |
+--------------------------------------------+--------------------------------------------------------------------+
                                             | I2C1 (SCL, SDA) + ALERT Interrupt
                                             v
                     +------------------------------------------------+
                     |       STM32L431CBT6 MCU (ARM Cortex-M4)        |
                     |   Reads 12x INA260, Packets CAN Frames         |
                     +-----------------------+------------------------+
                                             | bxCAN
                                             v
                     +------------------------------------------------+
                     |    TI TCAN1042V-Q1 Transceiver + TVS + Term    |
                     +-----------------------+------------------------+
                                             | CAN_H / CAN_L (500 kbps, IDs 0x380 - 0x383)
                                             v
+-----------------------------------------------------------------------------------------------------------------+
|                        TE CONNECTIVITY AMPSEAL CONNECTOR (Vehicle Output Interface)                             |
|                            Builds directly upon last year's pinout (PCB_FUSIBLES_XT60)                          |
+-----------------------------------------------------------------------------------------------------------------+
```

---

## 3. Hardware & Design Requirements

### 3.1 Core Component & Electrical Requirements
- **Low-Voltage Power Input & Copper Pours:** Molex Mega-Fit 4-circuit vertical header (Molex P/N: 172065-0004). Positive locking latch, polarized keying, rated up to 23A/contact (46A aggregate continuous capacity on dual 24V/GND circuits). On the PCB, the ground terminals pour directly into an unbroken solid ground plane covering the entire board, while the +24V supply terminals pour into a dedicated high-current supply plane making direct continuous contact with the line-side terminals of all 12 Keystone fuseholders.
- **12 Fused Channels & Blown Fuse Circuitry:** Twelve (12) Keystone 3568 automotive mini blade fuse clip pairs. Equipped with anti-parallel 0805 red indicator LEDs with 4.7 kΩ current-limiting resistors for instant visual inspection of blown fuses.
- **Current & Voltage Sensing:** Twelve (12) Texas Instruments INA260AIPWR current, voltage, and power monitor ICs with integrated 2 mΩ precision shunts. All 12 ICs share a single I2C bus using distinct hardware addresses (`0x40` to `0x4B`).
- **Power Conversion Subsystem:** Four independent DC-DC conversion stages from 24V raw accumulator bus:
  1. Murata OKI-78SR-3.3/1.5-W36-C for logic;
  2. Mornsun K7812-2000R3 or TI LMR33630 for DTI Inverter & Motor Duo (12V @ 2A);
  3. Murata OKI-78SR-12/1.0-W36-C for isolated Optical LiDAR (12V @ 10W);
  4. High-efficiency synchronous buck converter (TI LM5145RGYR + dual CSD18534Q5A MOSFETs) or 100W PoL brick for LattePanda Mission Computer (19V @ 90W continuous).
- **Processing Engine:** STM32L431CBT6 ARM Cortex-M4 microcontroller in LQFP-48 package running at 80 MHz with 128 KB Flash, 64 KB SRAM, and hardware bxCAN 2.0B.
- **CAN Physical Layer:** Texas Instruments TCAN1042V-Q1 (SOIC-8, AEC-Q100, 3.3V logic VIO, ±58V bus fault protection) with split 120 Ω termination (2x 60.4 Ω 1% + 4.7 nF to GND) and NUP2105L TVS diode protection.
- **Vehicle Interface Connector:** Sealed TE Connectivity AMPSEAL connector header providing vehicle power distribution, grounds, and CAN bus lines, designed to maintain backwards compatibility with existing car wiring.

### 3.2 Pin Mapping & Interface Specification

#### Table 3.2.1: 12-Channel Power Distribution & Allocation Matrix
| Ch # | Channel Net | Voltage Domain | Continuous / Peak | Fuse Rating | INA Addr | Primary Connected Subsystem |
| :---: | :--- | :---: | :---: | :---: | :---: | :--- |
| **01** | `12V_DTI_LOGIC` | 12V Regulated | 2.0 A / 3.0 A | 5A | `0x40` | DTI Inverter Logic & Motor Duo Control |
| **02** | `12V_LIDAR` | 12V Regulated | 0.83 A / 1.5 A | 3A | `0x41` | Driverless Optical LiDAR Sensor (10W Isolated) |
| **03** | `19V_LATTEPANDA` | 19V Regulated | 4.74 A / 6.0 A | 10A | `0x42` | LattePanda Mission Computer (90W Continuous) |
| **04** | `24V_ECU` | 24V Raw Bus | 2.0 A / 4.0 A | 5A | `0x43` | Main Vehicle Control Unit (IFS08-CE-ECU) |
| **05** | `24V_DASH_PDU` | 24V Raw Bus | 2.0 A / 4.0 A | 5A | `0x44` | Cockpit Dashboard Displays & PDU Distribution |
| **06** | `24V_SDC` | 24V Raw Bus | 1.5 A / 3.0 A | 5A | `0x45` | Vehicle Shutdown Circuit (SDC Master Loop) |
| **07** | `24V_TSAL` | 24V Raw Bus | 1.5 A / 3.5 A | 5A | `0x46` | Tractive System Active Light Controller |
| **08** | `24V_REFRI` | 24V Raw Bus | 8.0 A / 12.0 A | 15A | `0x47` | Radiator Cooling Fans & Water Pump Motors |
| **09** | `24V_ACU_BMS` | 24V Raw Bus | 2.0 A / 4.0 A | 5A | `0x48` | Accumulator Control Unit & High-Voltage BMS |
| **10** | `24V_DV_ACT` | 24V Raw Bus | 6.0 A / 10.0 A | 15A | `0x49` | Driverless Steering Actuator & Emergency Brake |
| **11** | `24V_AUX_TEL` | 24V Raw Bus | 2.0 A / 4.0 A | 5A | `0x4A` | Telemetry Ground Link Radio, GPS & IMU |
| **12** | `24V_SPARE` | 24V Raw Bus | 2.0 A / 4.0 A | 5A | `0x4B` | Configurable Auxiliary Expansion Rail |

#### Table 3.2.2: INA260 Digital Monitor Hardware Addressing (Shared I2C1 Bus @ 400 kHz)
| Channel # | Monitored Output Rail | Pin A1 Connection | Pin A0 Connection | 7-Bit I2C Address (Hex) |
| :---: | :--- | :---: | :---: | :---: |
| **CH 01** | `12V_DTI_LOGIC` | `GND` | `GND` | `0x40` |
| **CH 02** | `12V_LIDAR` | `GND` | `VS+ (3.3V)` | `0x41` |
| **CH 03** | `19V_LATTEPANDA` | `GND` | `SDA` | `0x42` |
| **CH 04** | `24V_ECU` | `GND` | `SCL` | `0x43` |
| **CH 05** | `24V_DASH_PDU` | `VS+ (3.3V)` | `GND` | `0x44` |
| **CH 06** | `24V_SDC` | `VS+ (3.3V)` | `VS+ (3.3V)` | `0x45` |
| **CH 07** | `24V_TSAL` | `VS+ (3.3V)` | `SDA` | `0x46` |
| **CH 08** | `24V_REFRI` | `VS+ (3.3V)` | `SCL` | `0x47` |
| **CH 09** | `24V_ACU_BMS` | `SDA` | `GND` | `0x48` |
| **CH 10** | `24V_DV_ACT` | `SDA` | `VS+ (3.3V)` | `0x49` |
| **CH 11** | `24V_AUX_TEL` | `SDA` | `SDA` | `0x4A` |
| **CH 12** | `24V_SPARE` | `SDA` | `SCL` | `0x4B` |

#### Vehicle Output Harness Integration Directive:
> [!IMPORTANT]
> **CRITICAL HARNESS BACKWARDS COMPATIBILITY REQUIREMENT:**  
> The output power distribution connector (TE Connectivity AMPSEAL) is not restricted to a rigid, new pinout schedule. Instead, **it must build directly upon last year's pinout (`PCB_FUSIBLES_XT60` v2)**.
> 
> **Engineering Implementation Rules:**
> 1. All existing vehicle output nets (ECU, SDC, TSAL, DASH, PDU, REFRI/Cooling, and ACU/BMS) must maintain their proven pin locations from the previous iteration, ensuring backwards compatibility with the existing vehicle harness.
> 2. The newly introduced regulated rails (12V DTI Inverter Logic, 12V LiDAR, and 19V LattePanda) alongside CAN_H and CAN_L communication lines must be allocated into the designated spare / auxiliary pin positions of the AMPSEAL header.
> 3. Multi-pin paralleling is preserved for high-current loads (such as Cooling Fans and LattePanda) to maintain contact resistance and thermal margins.

#### Table 3.2.3: CAN Telemetry Message Mapping (ECU Conflict-Free IDs: 0x380 to 0x383)
| CAN ID | Message Name | Cycle Rate | DLC | Contained Telemetry Signals |
| :---: | :--- | :---: | :---: | :--- |
| **0x380 (896)** | `SFUS_STATUS_SUMMARY` | 50 Hz (20 ms) | 8 | V_Bat_Raw (0.01V), I_Total (0.05A), Fuse_Status_Word (12-bit mask), PCB_Temp (°C), Heartbeat Counter |
| **0x381 (897)** | `SFUS_REGULATED_RAILS` | 10 Hz (100 ms) | 8 | I_DTI_Inv (1mA), I_LiDAR (1mA), I_LattePanda (2mA), V_LattePanda (0.1V), DC-DC Health Bitmask |
| **0x382 (898)** | `SFUS_CRITICAL_RAILS` | 10 Hz (100 ms) | 8 | I_ECU (1mA), I_Dash_PDU (1mA), I_SDC_Loop (1mA), I_TSAL (1mA) |
| **0x383 (899)** | `SFUS_HIGH_POWER_RAILS` | 10 Hz (100 ms) | 8 | I_Refri_Fans (10mA), I_ACU_BMS (1mA), I_DV_Actuators (5mA), I_Aux_Telemetry (1mA) |

> [!NOTE]
> **ECU CAN Compatibility Audit:**  
> A previous proposal evaluated CAN ID `0x4A0`, which directly collided with `AMS_status` (ID 1184 / 0x4A0) in `ecu.dbc`, read by the ECU's `VehicleService` task. The IDs are formally assigned to the **`0x380`–`0x383`** block (decimal 896–899), which is 100% free across all three ECU CAN buses (`FDCAN1` Inverter, `FDCAN2` ACU/Telemetry, `FDCAN3` Dashboard), providing guaranteed conflict-free arbitration and higher bus priority.

### 3.3 Testability & Mechanical Requirements
- **2-Layer PCB Stackup & Plane Architecture:** The PCB stackup is strictly specified as a **TWO-LAYER (2-Layer)** board utilizing heavy **2 oz (70 µm finished) copper on both Top and Bottom layers**.
  - **Bottom Layer:** Configured as an unbroken, continuous solid ground (GND) plane covering the entire board, ensuring minimal return loop inductance, excellent EMI shielding, and high thermal dissipation.
  - **Top Layer:** Features a dedicated, low-impedance +24V supply copper plane that pours directly from the Mega-Fit input terminals and runs across the board to make direct solid contact with the input terminals of all 12 Keystone 3568 mini blade fuseholders (acting as an integrated PCB busbar).
  - **Substrate:** High-Tg FR4 core (1.5 mm, $T_g \ge 150^\circ\text{C}$), resulting in standard 1.6 mm finished board thickness.
- **On-Board Test Points:** Dedicated test points on top copper with silkscreen labels: `TP_24V`, `TP_GND`, `TP_3V3`, `TP_12V_DTI`, `TP_12V_LIDAR`, `TP_19V_PANDA`, `TP_CAN_H`, `TP_CAN_L`, and `TP_SDA` / `TP_SCL`.
- **Mechanical Mounting:** Four (4) symmetric M3 mounting holes (3.20 mm drill diameter, 6.00 mm outer plated annular ring) located 5.0 mm from PCB corners to secure the board inside a custom sealed enclosure.
- **High-Current Trace Geometry:** IPC-2152 trace sizing ensures $\Delta T \le 20^\circ\text{C}$ on 15A continuous rails (5.0 mm width on 2 oz copper). 90W LattePanda rail requires minimum 2.5 mm width. Unregulated 40A input bus utilizes stitched polygon copper fills on top and bottom.
- **Pit-Stop Ergonomics:** All 12 Keystone fuse sockets aligned along one unobstructed PCB edge for rapid one-handed replacement in the paddock.

---

## 4. Verification & Testing Protocol

Every manufactured Smart Fusibles board must complete the four qualification testing phases in strict sequence:

### 4.1 Power Supply Rails Verification (TP_3V3, TP_12V_DTI, TP_12V_LIDAR, TP_19V_PANDA)
- **Procedure:** Before applying power, verify resistance from `+24V_IN` to `POWER_GND` > 100 kΩ, and all output pins to `GND` > 50 kΩ using a calibrated DMM. Connect 24.0V from a current-limited benchtop DC supply.
- **Acceptance Criteria:** `TP_3V3` must measure 3.30V ± 0.08V (ripple < 25 mVp-p); `TP_12V_DTI` must measure 12.00V ± 0.25V (ripple < 35 mVp-p); `TP_12V_LIDAR` must measure 12.00V ± 0.25V (ripple < 30 mVp-p); `TP_19V_PANDA` must measure 19.00V ± 0.35V (ripple < 50 mVp-p). Quiescent unpopulated current < 45 mA at 24V.

### 4.2 I2C Sensor Bus & Microcontroller Enumeration Test
- **Procedure:** Flash I2C bus diagnostic scanner firmware via the SWD header using an ST-Link V3 debugger.
- **Acceptance Criteria:** All 12 INA260 I2C addresses (`0x40` through `0x4B`) must acknowledge (ACK) within 50 ms. Voltage telemetry accuracy must be within 0.5% of DMM bench readings.

### 4.3 CAN Telemetry Verification with ECU & Vehicle Bus
- **Procedure:** Connect a CAN bus analyzer (PCAN-USB or Kvaser) at 500 kbps, 8-N-1. Verify split termination resistance across `CAN_H` and `CAN_L`.
- **Acceptance Criteria:** Split termination measures 60.0 Ω ± 2.0 Ω. Continuous transmission of frames `0x380`, `0x381`, `0x382`, and `0x383` at 50 Hz and 10 Hz with 0% bus error rate over 15 minutes of continuous operation. No collisions with ECU or AMS traffic.

### 4.4 Full 90W Thermal Soak & Blown Fuse Simulation Test
- **Procedure:** Apply an electronic DC load to `OUT_CH03_19V_PANDA` stepped to 4.74 A (90W continuous) for 30 minutes. Sequentially pull each of the 12 mini blade fuses under a 100 mA simulated load.
- **Acceptance Criteria:** Maximum regulator and inductor surface temperature remains < 75°C under thermal imaging on the 2-layer board. Each blown fuse illuminates its corresponding red hardware indicator LED instantly and sets its respective bit in the `SFUS_Fuse_Status_Word` (CAN `0x380`) within 20 ms.

---

## 5. Quality Assurance Test Execution Sign-Off Sheet

| Test Item | Target / Acceptance Criteria | Measured Value | Pass / Fail | Inspector | Date |
| :---: | :--- | :---: | :---: | :---: | :---: |
| **1. Unpowered Isolation** | $R(24\text{V to GND}) > 100\text{ k}\Omega$; Outputs $> 50\text{ k}\Omega$ | | | | |
| **2. 3.3V Logic Rail** | $3.22\text{V} – 3.38\text{V}$ at `TP_3V3`; Ripple $< 25\text{ mV}$ | | | | |
| **3. 12V DTI Rail** | $11.75\text{V} – 12.25\text{V}$ at `TP_12V_DTI` @ 2.0A | | | | |
| **4. 12V LiDAR Rail** | $11.75\text{V} – 12.25\text{V}$ at `TP_12V_LIDAR` @ 0.83A | | | | |
| **5. 19V LattePanda Rail** | $18.65\text{V} – 19.35\text{V}$ at `TP_19V_PANDA` @ 4.74A | | | | |
| **6. I2C Bus Scan (12x INA)** | All 12 Addresses (`0x40`–`0x4B`) ACK | | | | |
| **7. CAN Termination & Wave** | $R_{term} = 60.0\text{ }\Omega \pm 2\Omega$; clean eye diagram | | | | |
| **8. CAN Telemetry Stream** | Frames `0x380`–`0x383` streaming; 0 errors | | | | |
| **9. Blown Fuse Detection** | 12/12 channels trigger visual LED & CAN flag | | | | |
| **10. 90W Thermal Soak (2-Layer)** | $T_{max} < 85^\circ\text{C}$ after 30 min @ 90W continuous | | | | |

---

## 6. Post-Design Requirement: Design Report

Upon completion of the PCB layout, hardware fabrication, and bench/vehicle testing, a comprehensive Design Report document must be filled out and submitted for technical review.

The Design Report must document:
- **Schematic & Layout Documentation:** Detailed schematic diagrams, 2-layer 2 oz stackup, high-current trace routing, and copper ground pour implementation.
- **Bill of Materials (BOM):** Complete list of manufacturer part numbers, exact SMD/through-hole footprints, and supplier sourcing (LCSC/Mouser/DigiKey).
- **3D Mechanical Integration:** Sealed enclosure integration, AMPSEAL and Mega-Fit port clearances, and M3 insert stress considerations.
- **Validation Test Results:** Oscilloscope captures, measured voltage rails, CAN bus logs, and thermal imaging captures at 90W load.
- **Design Iterations & Recommendations:** Assembly issues encountered, thermal behavior on 2-layer copper, and proposed revisions for next-generation hardware.
