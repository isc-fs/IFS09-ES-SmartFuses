<img width="470.235" height="179.4" alt="isc-full-primary" src="https://github.com/user-attachments/assets/31365569-11bf-427e-ae3e-8d81ca87d765" />

# IFS09-ES-SmartFuses

Hardware design, KiCad schematics, PCB layout, multi-rail DC-DC power conversion, and CAN telemetry firmware for the **Smart Fusibles Board** of the **IFS09**, developed for the **ISC Formula Student Racing Team** (Season 2026/2027).

[![Formula Student](https://img.shields.io/badge/Formula%20Student-ISC-yellow.svg)](https://www.comillas.edu/)
[![Season](https://img.shields.io/badge/Season-2026%2F2027-blue.svg)]()
[![Hardware](https://img.shields.io/badge/EDA-KiCad%209.x%2F10.x-orange.svg)]()
[![Microcontroller](https://img.shields.io/badge/MCU-STM32L431CBT6%20(ARM%20Cortex--M4)-green.svg)]()
[![Sensors](https://img.shields.io/badge/Sensors-12x%20TI%20INA260%20(I2C)-purple.svg)]()
[![CAN Bus](https://img.shields.io/badge/CAN-500%20kbps%20(TCAN1042V--Q1)-red.svg)]()

[![View Work Order PDF](https://img.shields.io/badge/Work_Order-WO__ES--IFS09-blue?logo=adobeacrobatreader)](WO_ES-IFS09-SmartFuses.pdf)
[![Download Word Document](https://img.shields.io/badge/Work_Order-DOCX-blue?logo=microsoftword)](WO_ES-IFS09-SmartFuses.docx)
[![CAN DBC File](https://img.shields.io/badge/CAN_DBC-SMART__FUSIBLES-orange?logo=vector)](docs/dbc/SMART_FUSIBLES.dbc)

---

## Getting started

1. Create a GitHub account if you don't have one yet.
2. Download and install [GitHub Desktop](https://desktop.github.com/) (beginner) or [Git CLI](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) (advanced).
   - If this is your first time using GitHub Desktop, make sure to read the [User Manual](https://help.github.com/desktop/guides/).
   - If this is your first time using Git, start with a tutorial:
     - [Git Tutorial](https://git-scm.com/docs/gittutorial)
     - [Atlassian Git Tutorial](https://www.atlassian.com/git/tutorials/)
   - Keep a copy of [GitHub's Git Cheat Sheet](https://services.github.com/kit/downloads/github-git-cheat-sheet.pdf) handy as a reference.

3. Clone this repository to your machine:
   - **SSH:** `git@github.com:isc-fs/IFS09-ES-SmartFuses.git`
   - **HTTPS:** `https://github.com/isc-fs/IFS09-ES-SmartFuses.git`

---

## How we work with this repository

### Main branches

The repository follows standard ISC branching rules:

* **`main`** is the production branch. It contains only validated hardware releases, schematics, and documentation ready for track operations. Never work directly on it.
* **`dev`** (or active feature branches) is the development and integration point. Never commit directly to `main` — all changes arrive through a reviewed Pull Request.

```
main  ──────────────────●──────────────────────●──▶  (validated releases only)
                        ↑                      ↑
dev   ──────●───●───●───●───●───●───●───●───●──●──▶  (continuous integration)
            ↑   ↑       ↑   ↑   ↑       ↑   ↑
          feat/1 fix/1 feat/2 fix/2   feat/3 fix/3
```

### Feature branches

All work — whether a new hardware revision, firmware update, or bug fix — is done on a **feature branch** created from `dev` (or `main`). When the work is ready, a Pull Request is opened, reviewed, merged, and the branch is deleted.

```
feat/<n>   →  new functionality / hardware addition (feat/1, feat/2, feat/3 ...)
fix/<n>    →  bug fix / schematic correction       (fix/1,  fix/2,  fix/3  ...)
```

### Step-by-step workflow

1. **Create the branch:**
   ```bash
   git checkout main
   git pull origin main
   git checkout -b feat/your-feature-name
   ```
2. **Work and commit:**
   ```bash
   git add .
   git commit -m "feat(hw): add reverse polarity protection to 24V input"
   ```
3. **Push and open Pull Request:**
   ```bash
   git push origin feat/your-feature-name
   ```
   Open a PR against `main` (or `dev`), verify 0 KiCad DRC/ERC errors, provide test evidence, and request review from the Electronics Subsystem Technical Director.

---

## 1. Project Overview & System Purpose

The **IFS09 Smart Fusibles Board** acts as the primary low-voltage (LV) power distribution, multi-rail DC-DC conversion, circuit protection, and digital power telemetry hub for the IFS09 race car.

It evolves the previous passive revision (`PCB_FUSIBLES_XT60` v2) into an active, intelligent subsystem that:
1. Replaces non-locking friction-fit XT-60 connectors with an automotive positive-locking **Molex Mega-Fit** 4-pin header.
2. Protects **12 independent power channels** using **Keystone 3568** automotive mini blade fuse clips with dual-tier blown-fuse detection (instant visual hardware LEDs + CAN telemetry flags).
3. Monitors real-time voltage, current, and power on every branch via **12 Texas Instruments INA260AIPWR** sensors residing on a single shared 400 kHz I2C bus via unique hardware addressing (`0x40` to `0x4B`).
4. Generates onboard regulated power rails directly from the 24V battery: **3.3V logic**, **12V @ 2A** (DTI Inverter logic & motor duo), **12V @ 10W** (isolated Driverless Optical LiDAR), and **19V @ 90W** (LattePanda Mission Computer).
5. Transmits live telemetry to the vehicle ECU and ground station at 50 Hz / 10 Hz over CAN using conflict-free message IDs **`0x380`–`0x383`**.

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

## 2. Hardware Architecture & Electrical Specifications

### 2.1 12 Protected Fused Power Channels
All 12 channels utilize **Keystone 3568** through-hole automotive mini blade fuse clip pairs.

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

* **Dual-Tier Blown-Fuse Detection:**
  1. *Hardware Visual Tier:* Reverse-biased red 0805 SMD LED in series with a 4.7 kΩ 1/4W resistor across each fuse clip. Normally shorted by the fuse (OFF); turns ON immediately when the fuse blows.
  2. *Software Telemetry Tier:* Downstream INA260 registers bus voltage drop to 0V while raw 24V bus remains active, setting the channel bit in the `SFUS_Fuse_Status_Word` over CAN within 20 ms.

### 2.2 Digital Power Telemetry: 12x TI INA260AIPWR
* **Integrated Precision Shunt:** Built-in 2.0 mΩ (0.1% tolerance, 15 ppm/°C) shunt resistor eliminates 12 large 2512 SMD power resistors and Kelvin sensing routing errors.
* **Single Shared I2C Bus:** Pins `A0` and `A1` are pin-strapped to GND, 3.3V, SDA, or SCL to configure 12 unique 7-bit addresses (`0x40` to `0x4B`), allowing all 12 devices to share `I2C1` with zero multiplexers.
* **36V Common-Mode Margin:** Rated for 0V to 36V continuous (40V abs max), safely withstanding 24V automotive inductive spikes and load dumps.

### 2.3 Integrated Multi-Rail DC-DC Power Conversion
* **3.3V Logic Rail:** Murata **OKI-78SR-3.3/1.5-W36-C** (SIP-3, 36V in, 1.5A out, >85% eff.) supplying STM32L4, TCAN1042V $V_{IO}$, and INA logic.
* **12V @ 2A (24W) - DTI Inverter & Motor Duo:** Mornsun **K7812-2000R3** (SIP-3 module) or TI **LMR33630** (36V, 3A sync buck IC). Output ripple < 40 mVp-p.
* **12V @ 10W (0.83A) - Driverless LiDAR:** Murata **OKI-78SR-12/1.0-W36-C** (SIP-3 module), completely isolating optical LiDAR from motor actuator switching noise.
* **19V @ 90W (4.74A) - LattePanda Mission Computer:** High-efficiency synchronous step-down converter based on the **TI LM5145RGYR** controller with dual **TI CSD18534Q5A** 60V 7.8 mΩ MOSFETs and a 6.8 µH 10A inductor (or high-density industrial 100W PoL module like Cincon EC7BW18-72S19).

### 2.4 Automotive Locking Input: Molex Mega-Fit
* **Part Number:** Molex **172065-0004** (Vertical Header, 4 circuits, 5.70 mm pitch) with positive locking latch and terminal position assurance (TPA).
* **Current Rating:** 23.0 A per terminal. With dual-pin paralleling (Pins 1–2 for `+24V_IN`, Pins 3–4 for `POWER_GND`), the aggregate continuous capacity is **46.0 A**.

### 2.5 Vehicle Output Interface: TE Connectivity AMPSEAL
> [!IMPORTANT]
> **Harness Backwards Compatibility Directive:**  
> The output connector pinout is not restricted to a rigid, new schedule. Instead, **it builds directly upon last year's pinout (`PCB_FUSIBLES_XT60` v2)**. All proven vehicle branch assignments (ECU, SDC, TSAL, DASH, PDU, REFRI/Cooling, and ACU/BMS) are preserved for backwards compatibility with the vehicle harness, while the new regulated rails (12V DTI, 12V LiDAR, 19V LattePanda) and CAN lines occupy the spare/expansion pins.

---

## 3. PCB Layout, Mechanical & Copper Plane Architecture

### 3.1 2-Layer Stackup & Copper Plane Strategy
* **Layer Count:** Strictly specified as a **TWO-LAYER (2-Layer)** FR4 board ($T_g \ge 150^\circ\text{C}$).
* **Copper Weight:** **2 oz (70 µm finished) copper on both Top and Bottom layers**. Total thickness: 1.6 mm.
* **Bottom Layer (Full-Board GND Plane):**
  - Configured as an unbroken, continuous solid ground (GND) reference plane covering the entire board.
  - Ground pins (Pins 3 & 4) of the Molex Mega-Fit connector pour directly into this plane.
  - High-density thermal/ground via stitching provides an ultra-low impedance return path for high-current motor/fan loads while shielding sensitive digital lines (I2C, CAN, SWD).
* **Top Layer (+24V Supply Bus Plane):**
  - Features a dedicated, low-impedance +24V supply copper plane zone that pours directly from the Mega-Fit input terminals through the TVS/protection stage.
  - Runs across the board to make direct solid contact with the line-side terminals of all 12 Keystone 3568 mini blade fuseholders, functioning as an integrated PCB busbar that eliminates trace necking and minimizes $IR$ drop under 40A+ aggregate vehicle loads.
* **Mechanical Mounting:** Four (4) symmetric **M3 mounting holes** (3.20 mm drill diameter, 6.00 mm outer plated annular ring) placed 5.0 mm from PCB corners.

---

## 4. CAN Telemetry Data Dictionary & ECU Compatibility

The STM32L4 samples all 12 INA260 channels via DMA and broadcasts telemetry frames over the vehicle CAN bus at 500 kbps (Intel Little-Endian format).

> [!NOTE]
> **ECU CAN Compatibility Audit:**  
> The initially evaluated CAN ID `0x4A0` (decimal 1184) collided directly with `AMS_status` in `ecu.dbc`, which is actively decoded by the ECU's `can_rx_task.cpp` for battery safety. The IDs are formally assigned to the **`0x380`–`0x383`** block (decimal 896–899), which is 100% free across all three ECU CAN buses (`FDCAN1` Inverter, `FDCAN2` ACU/Telemetry, `FDCAN3` Dashboard) and grants higher arbitration priority for critical blown-fuse alerts.

| CAN ID | Message Name | Cycle Rate | DLC | Contained Telemetry Signals |
| :---: | :--- | :---: | :---: | :--- |
| **0x380 (896)** | `SFUS_STATUS_SUMMARY` | 50 Hz (20 ms) | 8 | V_Bat_Raw (0.01V), I_Total (0.05A), Fuse_Status_Word (12-bit mask), PCB_Temp (°C), Heartbeat Counter |
| **0x381 (897)** | `SFUS_REGULATED_RAILS` | 10 Hz (100 ms) | 8 | I_DTI_Inv (1mA), I_LiDAR (1mA), I_LattePanda (2mA), V_LattePanda (0.1V), DC-DC Health Bitmask |
| **0x382 (898)** | `SFUS_CRITICAL_RAILS` | 10 Hz (100 ms) | 8 | I_ECU (1mA), I_Dash_PDU (1mA), I_SDC_Loop (1mA), I_TSAL (1mA) |
| **0x383 (899)** | `SFUS_HIGH_POWER_RAILS` | 10 Hz (100 ms) | 8 | I_Refri_Fans (10mA), I_ACU_BMS (1mA), I_DV_Actuators (5mA), I_Aux_Telemetry (1mA) |

---

## 5. Formal Design Documents & Reports

The project specification and verification criteria are documented across the following official engineering artifacts:

| Document | File Link | Description | Status |
| :--- | :--- | :--- | :--- |
| **Work Order (PDF)** | [📄 WO_ES-IFS09-SmartFuses.pdf](WO_ES-IFS09-SmartFuses.pdf) | Formal published Work Order specification generated using the official ISC team base template (`IFS_09_W_Template_Base_CB.docx`). | **Rev 1.0 (Released)** |
| **Work Order (DOCX)** | [📄 WO_ES-IFS09-SmartFuses.docx](WO_ES-IFS09-SmartFuses.docx) | Microsoft Word source document with native Jost / IBM Plex Sans typography, cover page, and ISC table styles. | **Rev 1.0 (Released)** |
| **CAN Database** | [📄 SMART_FUSIBLES.dbc](docs/dbc/SMART_FUSIBLES.dbc) | Vector CAN database defining messages `0x380`–`0x383` ready for ECU integration and CANoe/PCAN logging. | **v1.0 (Active)** |
| **Markdown Specification** | [📄 WORK_ORDER_SMART_FUSIBLES.md](docs/WORK_ORDER_SMART_FUSIBLES.md) | Synchronized Markdown specification for Git version control and quick online reference. | **Rev 1.0** |

---

## 6. Repository Structure

```
IFS09-ES-SmartFuses/
├── docs/                              # Documentation & Database Assets
│   ├── dbc/
│   │   └── SMART_FUSIBLES.dbc         # Official Vector CAN DBC File
│   ├── images/                        # Architecture & PCB Render Graphics
│   └── WORK_ORDER_SMART_FUSIBLES.md   # Synchronized Markdown Specification
├── IFS09-SmartFuses/                  # KiCad Project Files (KiCad 9.x / 10.x)
│   ├── 3568_Fuse_Holder/              # Keystone 3568 Footprint, Symbol & 3D STEP
│   ├── fp-lib-table                   # Footprint Library Table
│   ├── sym-lib-table                  # Symbol Library Table
│   ├── SMART_FUSIBLES.dbc             # Local DBC Mapping File
│   └── IFS09-SmartFuses.kicad_pro     # KiCad 2-Layer 2 oz Project File
├── WO_ES-IFS09-SmartFuses.pdf         # Official Work Order (Vector PDF Export)
├── WO_ES-IFS09-SmartFuses.docx        # Official Work Order (Base Template Word Doc)
├── .gitignore                         # KiCad and OS Ignore Rules
└── README.md                          # Repository Documentation & Architecture Guide
```

---

## 7. Validation & Testing Protocol

Upon fabrication of PCB units, the following sequential validation protocol will be executed:

```mermaid
flowchart LR
    A[1. Power Rails (TP_3V3/12V/19V)] --> B[2. I2C Bus Scan (12x INA)]
    B --> C[3. CAN Stream (0x380-0x383)]
    C --> D[4. 90W Thermal Soak & Fuse Sim]
```

1. **Power Supply Rails Verification:**
   * Unpowered resistance check: `+24V_IN` $\rightarrow$ `GND` ($> 100\,\text{k}\Omega$); Output pins $\rightarrow$ `GND` ($> 50\,\text{k}\Omega$).
   * Connect 24.0V bench supply: Verify `TP_3V3` ($3.30\,\text{V} \pm 0.08\,\text{V}$), `TP_12V_DTI` ($12.00\,\text{V} \pm 0.25\,\text{V}$), `TP_12V_LIDAR` ($12.00\,\text{V} \pm 0.25\,\text{V}$), and `TP_19V_PANDA` ($19.00\,\text{V} \pm 0.35\,\text{V}$). Ripple $< 40\,\text{mV}_{p-p}$.
2. **I2C Sensor Bus Enumeration:**
   * Flash diagnostic scanner over SWD: Confirm all 12 INA260 addresses (`0x40`–`0x4B`) acknowledge (ACK) within 50 ms.
   * Telemetry voltage reading accuracy within 0.5% of DMM bench readings.
3. **CAN Physical Layer & Telemetry Stream:**
   * Verify split termination measures $60.0\,\Omega \pm 2.0\,\Omega$ across `CAN_H` and `CAN_L`.
   * Continuous streaming of frames `0x380`–`0x383` at 500 kbps with 0% bus errors over 15 minutes. No collisions with vehicle ECU traffic.
4. **90W Thermal Soak & Blown-Fuse Simulation:**
   * Apply 4.74 A (90W continuous) electronic load to `OUT_CH03_19V_PANDA` for 30 minutes; verify component temperatures remain $< 75^\circ\text{C}$ on 2-layer copper.
   * Pull each mini blade fuse under 100 mA load; confirm immediate red LED illumination and corresponding CAN bitmask flag within 20 ms.

---

## 8. Author & Team Credits

* **Subsystem:** Low Voltage & Electronics Subsystem (ES)
* **Author & Technical Lead:** Andrés Sánchez de Ágreda
* **Team:** [ISC Formula Student Racing Team](https://www.comillas.edu/) — Universidad Pontificia Comillas (ICAI)
* **Season:** 2026 / 2027

---

*ISC Racing Team — IFS09 Electronics Subsystem*