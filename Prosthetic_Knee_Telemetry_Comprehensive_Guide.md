# Prosthetic Knee Telemetry System: Comprehensive Technical Reference Manual

> **Document Type:** Detailed Engineering Explanation & Companion Reference to Work Progress Presentation  
> **Author:** Jithin Sreenu  
> **Date:** November 2026  
> **Firmware Target:** STMicroelectronics B-U585I-IOT02A (ARM Cortex-M33 @ 160 MHz, FreeRTOS)  
> **Software Target:** Prosthetic Limb Telemetry Studio v1.3.0 (Electron 31 / Node 24 / Chromium)  
> **Project Scope:** Closed-loop biomechanical sensing, real-time gait phase classification, 100 Hz encrypted telemetry streaming, automated clinical analytics, and in-situ calibration.

---

## Table of Contents
1. [Executive Summary & High-Level System Motivation](#1-executive-summary--high-level-system-motivation)
2. [End-to-End System Architecture](#2-end-to-end-system-architecture)
3. [Embedded Hardware Platform & Peripheral Configuration](#3-embedded-hardware-platform--peripheral-configuration)
4. [Firmware Sensor Acquisition Pipeline & Filter Design](#4-firmware-sensor-acquisition-pipeline--filter-design)
5. [FreeRTOS Architecture & Multi-Tasking Design](#5-freertos-architecture--multi-tasking-design)
6. [Clinical Gait Intelligence: The 6-Phase State Machine](#6-clinical-gait-intelligence-the-6-phase-state-machine)
7. [Motor-Control Slave Link Protocol & Actuator Driving](#7-motor-control-slave-link-protocol--actuator-driving)
8. [Telemetry Protocols: Fixed 25-Byte v1 Frame vs 31-Byte Encrypted v2 Frame](#8-telemetry-protocols-fixed-25-byte-v1-frame-vs-31-byte-encrypted-v2-frame)
9. [Cryptographic Security: Hardware AES-256-GCM Implementation](#9-cryptographic-security-hardware-aes-256-gcm-implementation)
10. [Fault Diagnostics & 16-Bit Safety Register](#10-fault-diagnostics--16-bit-safety-register)
11. [In-Situ Calibration, Thresholds & Internal Flash Bank 2 Persistence](#11-in-situ-calibration-thresholds--internal-flash-bank-2-persistence)
12. [Desktop Studio Architecture (Electron Multi-Process Model)](#12-desktop-studio-architecture-electron-multi-process-model)
13. [Detailed Guide to All 16 Desktop Studio GUI Provisions](#13-detailed-guide-to-all-16-desktop-studio-gui-provisions)
14. [Engineering Findings, Discrepancies & Root-Cause Fixes](#14-engineering-findings-discrepancies--root-cause-fixes)
15. [Verification Status Matrix & Bench Test Results](#15-verification-status-matrix--bench-test-results)
16. [Strategic Roadmap & Recommended Next Steps](#16-strategic-roadmap--recommended-next-steps)
17. [Complete Technical Glossary](#17-complete-technical-glossary)

---

## 1. Executive Summary & High-Level System Motivation

### 1.1 The Clinical Problem
Above-knee (transfemoral) amputees rely on prosthetic knee joints to restore natural, symmetrical bipedal locomotion. Traditional mechanical knees utilize fixed passive friction or passive fluid resistance; they cannot adapt to changing walking cadences, slopes, or terrain transitions. When an amputee's heel strikes the ground, an unyielding or improperly damped knee will buckle, leading to catastrophic falls. Conversely, when the foot lifts into the air during the swing phase, excessive resistance forces the patient to swing their hip outward (hip circumduction) or vault on their sound limb, causing chronic spine and hip degeneration.

### 1.2 The Technological Solution
This project provides a complete, high-performance, closed-loop cyber-physical system composed of:
1. **A Smart Prosthetic Knee Joint:** Equipped with dual ground reaction force load cells, a precision joint angle potentiometer, an onboard lithium-ion battery monitor, and a motorized hydraulic spool valve controlled via a slave microcontroller.
2. **High-Performance Embedded Firmware:** Running on an STMicroelectronics STM32U585 microcontroller (ARM Cortex-M33 @ 160 MHz), executing a register-level DMA acquisition engine, a 51-tap low-pass digital FIR filter, and a 6-phase clinical gait state machine that actively commands the hydraulic spool valve at 100 Hz.
3. **Prosthetic Telemetry Studio (Electron Desktop Application):** A cross-platform clinical workstation providing real-time multi-pane waveform visualization, phosphor-glow oscilloscope rendering, interactive 3D WebGL phase-space trajectories, automated gait symmetry analytics, low-level binary protocol inspection, and in-situ calibration persistence.

### 1.3 The Core Analogy for First-Time Observers
To understand this system without prior biomedical engineering background:
> **Think of this system as an "Advanced Intensive Care Patient Monitor" built specifically for an amputee's robotic knee.**  
> Just as an ECG monitor continuously tracks heart waveforms, detects arrhythmias, and logs cardiac events, the STM32U585 microcontroller monitors joint stresses and angles 100 times per second, detects walking phases and mechanical anomalies, and automatically adjusts the hydraulic knee's damping so the patient never falls. Simultaneously, it broadcasts clinical telemetry to the doctor's screen via high-speed USB or encrypted Bluetooth.

---

## 2. End-to-End System Architecture

The complete system spans three distinct functional tiers: the **Sensing & Actuation Layer**, the **Embedded Processing Layer**, and the **Clinical Visualization Tier**.

```
+---------------------------------------------------------------------------------------------------+
|                                      1. PHYSICAL SENSING LAYER                                    |
|  [Load Cell 1: PC0/A0]   [Load Cell 2: PC2/A1]   [Knee Pot: PC4/A2]   [Battery Divider: 4.2V]    |
+------------------------------------------+--------------------------------------------------------+
                                           | Analog Voltages (0 - 3.3V)
                                           v
+---------------------------------------------------------------------------------------------------+
|                               2. EMBEDDED CONTROLLER (STM32U585)                                  |
|  +---------------------------------------------------------------------------------------------+  |
|  | ADC1 + GPDMA1 Circular Scan (3 Channels x 32 Samples = 96 Words/Block @ ~2,778 Blocks/sec)  |  |
|  +---------------------------------------------+-----------------------------------------------+  |
|                                                | Filtered Average
|                                                v
|  +---------------------------------------------------------------------------------------------+  |
|  | 51-Tap Hamming Low-Pass FIR Filter (25 Hz Cutoff) -> Calibration Scaling (N, Nm, Deg, V)    |  |
|  +---------------------------------------------+-----------------------------------------------+  |
|                                                | Engineering Units
|                                                v
|  +---------------------------------------------------------------------------------------------+  |
|  | 6-Phase Clinical Gait State Machine (state_machine.c) [mid_seen Sequencing Flag]            |  |
|  +----------------------+--------------------------------------+-------------------------------+  |
|                         | State Transition (0/1)               | 100 Hz Telemetry Packet (Values)|
|                         v                                      v                                  |
|  +------------------------------------+  +-----------------------------------------------------+  |
|  | USART3 Slave Link (slave_comm.c)   |  | Smart Protocol Builder (smart_protocol.c)           |  |
|  | 7-Byte Motor Frames (PD8/PD9)      |  | 25-Byte Frame | Little-Endian | CRC-16/MODBUS       |  |
|  | Median-of-3 Filter & 25ms Freshness|  +-----------------------------+-----------------------+  |
|  +------------------+-----------------+                                | 25-Byte Plaintext Frame  |
|                     |                                                  v                          |
|                     | Valve Feedback (%) +-----------------------------------------------------+  |
|                     +------------------->| Opt-in AES-256-GCM Hardware Encryption (crypto.c)   |  |
|                                          | 31-Byte v2 Frame: Header + Ciphertext + 8-Byte Tag  |  |
|                                          +-----------------------------+-----------------------+  |
|                                                                        | DMA TX Stream            |
+------------------------------------------------------------------------+--------------------------+
                                                                         | USART1 @ 115200 Baud
                                                                         v
+---------------------------------------------------------------------------------------------------+
|                        3. PROSTHETIC TELEMETRY STUDIO (ELECTRON DESKTOP APP)                      |
|  +---------------------------------------------------------------------------------------------+  |
|  | Main Process: Node-SerialPort / Web Serial / Web Bluetooth NUS -> IPC Bridge (contextBridge)|  |
|  +---------------------------------------------+-----------------------------------------------+  |
|                                                | Decoded 100 Hz Stream
|                                                v
|  +---------------------------------------------------------------------------------------------+  |
|  | Renderer Process: 20 Hz Batch Flush Ring Buffer -> 16 Rich Clinical & Engineering Tabs      |  |
|  | • Overview Dashboard (11 KPIs)           • PedoGraph Phosphor Oscilloscope (Canvas glow)   |  |
|  | • 4-Pane Waveforms (Plotly.js)           • 3D WebGL State Space Trajectory                 |  |
|  | • Clinical Gait Analytics (Cadence, ROM) • Frame Inspector (Color-coded Hex Grid)          |  |
|  | • Safety Thresholds & Calibration Upload • Local Session Recording (CSV/JSON Export)       |  |
|  +---------------------------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------------------------+
```

---

## 3. Embedded Hardware Platform & Peripheral Configuration

### 3.1 Microcontroller Specifications
The system targets the **STMicroelectronics B-U585I-IOT02A Discovery Board**, incorporating the advanced **STM32U585AII6Q** microcontroller:
* **Core:** ARM 32-bit Cortex-M33 CPU with Armv8-M architecture, TrustZone security, and single-precision hardware Floating Point Unit (FPU).
* **Clock Frequency:** Configured to run at **160 MHz** using the high-frequency PLL driven by the multi-speed internal (MSI) oscillator.
* **SRAM Configuration:** Total **768 KB of SRAM**, subdivided into SRAM1 (192 KB), SRAM2 (64 KB), SRAM3 (512 KB), and backup SRAM (2 KB).
* **Non-Volatile Memory:** **2 MB of Dual-Bank Flash**. Bank 2 is leveraged for zero-wear persistent parameter storage (coefficients, safety thresholds, and 256-bit cryptographic keys).
* **Operating Voltage:** 3.3V system rail with internal ultra-low-power linear voltage regulator (LDO) and switched-mode power supply (SMPS) step-down converter.

### 3.2 Pin Mapping & Peripheral Assignment Table
Based on the authoritative register-level source code (`main.c`, `adc.c`, `usart.c`, `gpdma.c`):

| Signal / Function | MCU Pin | Peripheral | Configuration / Mode | Electrical Destination / Sensor |
|---|---|---|---|---|
| **Load Cell 1 (Force)** | **PC0** | ADC1_IN1 | Regular Rank 1, 2.5 cycles | Arduino A0: Anterior Load Cell Strain Gauge |
| **Load Cell 2 (Moment)** | **PC2** | ADC1_IN3 | Regular Rank 2, 2.5 cycles | Arduino A1: Posterior Load Cell Strain Gauge |
| **Knee Joint Angle** | **PC4** | ADC1_IN13 | Regular Rank 3, 2.5 cycles | Arduino A2: Precision Rotary Potentiometer |
| **Expansion ADC 1** | PC5 | ADC1_IN14 | Rank 4 (Configured/Reserved)| Arduino A3: Future auxiliary channel |
| **Expansion ADC 2** | PA7 | ADC1_IN12 | Rank 5 (Configured/Reserved)| Arduino A4: Future auxiliary channel |
| **Expansion ADC 3** | PB0 | ADC1_IN15 | Rank 6 (Configured/Reserved)| Arduino A5: Future auxiliary channel |
| **Telemetry Host TX** | **PA9** | USART1_TX | 115200-8N1, GPDMA1 Ch0 | Virtual COM Port / ST-LINK to PC Host |
| **Telemetry Host RX** | **PA10** | USART1_RX | 115200-8N1, Idle-Line DMA | CLI Console / Command Parser from Host |
| **Motor Slave Link TX** | **PD8** | USART3_TX | 115200-8N1, Push-Pull | Inter-MCU link to Hydraulic Valve Slave |
| **Motor Slave Link RX** | **PD9** | USART3_RX | 115200-8N1, Idle-Line DMA | Valve Position Feedback (%) from Slave |
| **User Status LED** | PH7 | GPIO_Output | High-efficiency green LED | Heartbeat toggle on 100 Hz timer tick |
| **Hardware Randomness** | Internal| RNG | 32-bit TRNG peripheral | Entropy generation for session nonces |
| **Hardware Encryption** | Internal| AES | 256-bit block engine | Hardware AES round function acceleration |

---

## 4. Firmware Sensor Acquisition Pipeline & Filter Design

### 4.1 ADC Conversion Mechanics & Channel Scan
Sensor voltages are acquired through **ADC1** configured in 12-bit right-aligned resolution ($0 \text{ to } 4095$ counts for $0 \text{ to } 3.3\text{V}$):
$$\text{Voltage (V)} = \frac{\text{ADC Count}}{4095} \times V_{\text{REF}} \quad (V_{\text{REF}} = 3.3\text{V})$$

The ADC operates in continuous sequence scanning across the active regular ranks. Sampling time is set to $2.5 \text{ ADC clock cycles}$ ($15 \text{ total cycles per conversion}$). Powered from a dedicated 4 MHz MSI clock divider (MSIC), conversion of each channel takes $3.75\,\mu\text{s}$.

### 4.2 GPDMA1 Architecture & The Critical DMNGT Fix
Data transfer from the ADC data register (`ADC1->DR`) into memory is executed by **GPDMA1 Channel 1**. The buffer is dimensioned as:
$$\text{Buffer Size} = 32 \text{ scans} \times 3 \text{ channels} = 96 \text{ halfwords } (192 \text{ bytes})$$
This buffer is completed every $\sim 360\,\mu\text{s}$, producing an internal acquisition rate of $\sim 2,778 \text{ blocks per second}$.

> **The Root-Caused DMA Breakthrough:**  
> During initial hardware bring-up, real sensor readings on channels PC0 and PC2 remained zero while only battery voltage updated. Deep debugging identified that the STM32U585 ADC requires:
> ```c
> ADC1->CFGR1 = ADC_CFGR1_CONT | (0x02U << ADC_CFGR1_DMNGT_Pos);
> ```
> Setting `DMNGT = 10b` forces the ADC to issue a DMA request *after each and every channel conversion*. Furthermore, DMA re-arming was moved directly inside the `GPDMA1_Channel1_IRQHandler()` transfer-complete callback, completely eliminating a race condition between abort and restart.

### 4.3 51-Tap Low-Pass Digital FIR Filter
Prosthetic foot impacts generate aggressive mechanical vibrations and acoustic ring that could cause false state machine triggers. To eliminate this noise, the firmware runs a **51-tap symmetric Finite Impulse Response (FIR) low-pass filter** designed with a **Hamming window**:
* **Filter Order:** 50 ($M = 51 \text{ coefficients}$)
* **Cutoff Frequency ($f_c$):** $25\,\text{Hz}$
* **Sampling Frequency ($f_s$):** $100\,\text{Hz}$ (synchronous with the telemetry timer)
* **Mathematical Convolution:**
$$y[n] = \sum_{k=0}^{50} b[k] \cdot x[n-k]$$
Where $b[k]$ represents the pre-computed floating-point impulse response coefficients stored in `fir_filter.c`. This filter provides $> 40\,\text{dB}$ attenuation of high-frequency muscle tremor and ground impact noise while preserving true biological joint kinematic curves.

### 4.4 Engineering Units Conversion
Following filtering, normalized voltages are scaled into clinical units using calibrated scale factors:
* **Axial Ground Force ($F$):**
  $$F\,(\text{N}) = V_{\text{PC0}} \times 300.0$$
* **Sagittal Bending Moment ($M$):**
  $$M\,(\text{Nm}) = V_{\text{PC2}} \times 60.0$$
* **Knee Flexion Angle ($\theta$):**
  $$\theta\,(^\circ) = V_{\text{PC4}} \times 27.0$$
* **Battery Voltage ($V_{\text{bat}}$):**
  $$V_{\text{bat}}\,(\text{V}) = V_{\text{PC4\_alt}} \times 1.41 \quad (\text{resistor divider compensation})$$

---

## 5. FreeRTOS Architecture & Multi-Tasking Design

The firmware runs under the **FreeRTOS kernel** (CMSIS-RTOS v2 wrapper) to guarantee deterministic 100 Hz timing:

```
+---------------------------------------------------------------------------------------------------+
|                                      FREERTOS SCHEDULER (160 MHz)                                 |
+---------------------------------------------------------------------------------------------------+
       |
       +---> 1. High-Priority 100 Hz Software Timer (10 ms Period, TelemetryCallback)
       |     • Calls ADC_Poll() -> Reads latest DMA filtered buffer
       |     • Executes SM_Run() -> Evaluates 6-phase gait state machine & updates mid_seen flag
       |     • Checks Sensor Health -> Evaluates per-channel stuck-zero, stuck-VREF & flatline
       |     • Builds 25-Byte Packet -> Serializes fixed-point fields and computes CRC-16/MODBUS
       |     • Evaluates AES Encryption -> If armed, encrypts 20 bytes payload into 31-byte frame
       |     • Triggers GPDMA1 Ch0 -> Non-blocking DMA transfer to USART1 (drops frame if HAL_BUSY)
       |     • Toggles Green Heartbeat LED (PH7)
       |
       +---> 2. CLI Command Console Task (Normal Priority, USART1 RX Idle-Line DMA)
       |     • Circular buffer receives incoming ASCII commands from Desktop Studio
       |     • Parses: mode, demo, set, get, sensors, halt, sm, crypto, reboot
       |     • Dispatches responses asynchronously without stalling 100 Hz telemetry
       |
       +---> 3. Motor Slave Link Service (High Priority, USART3 RX Idle-Line DMA)
             • Processes incoming 7-byte valve feedback frames from hydraulic actuator
             • Applies median-of-3 filter on valve position (%)
             • Implements 25 ms freshness timeout gate
```

---

## 6. Clinical Gait Intelligence: The 6-Phase State Machine

### 6.1 State Machine Implementation (`state_machine.c`)
The gait state machine executes on every 10 ms timer cycle and classifies human movement into six distinct physiological phases:

```
                           +------------------------------------+
                           |             0: SWING               |
                           |   (Foot in air, Force < 0.2V)      |
                           +-----------------+------------------+
                                             | Heel Strike (Force rises)
                                             v
                           +------------------------------------+
                           |            2: LOADING              |
                           |   (Weight acceptance, mid_seen=0)  |
                           +-----------------+------------------+
                                             | Maximum Loading (Force > 0.8V)
                                             v
                           +------------------------------------+
                           |          3: MID-STANCE             |
                           | (Full single-leg stance, mid_seen=1|
                           +-----------------+------------------+
                                             | Body moves forward (Force drops)
                                             v
                           +------------------------------------+
                           |        4: TERMINAL-STANCE          |
                           | (Push-off, same force as loading   |
                           |        BUT mid_seen == 1)          |
                           +-----------------+------------------+
                                             | Unloading (Force < 0.4V)
                                             v
                           +------------------------------------+
                           |           5: PRE-SWING             |
                           | (Toe-off, knee flexing for swing)  |
                           +-----------------+------------------+
                                             | Foot leaves ground
                                             +----------> (Returns to 0: SWING)
```

### 6.2 The Phase Ambiguity Problem & The `mid_seen` Solution
In clinical gait analysis, **Phase 2 (Loading)** and **Phase 4 (Terminal-Stance)** generate nearly identical ground reaction force values ($300 \text{ to } 600\,\text{N}$). A purely threshold-based system cannot determine whether the patient is stepping down or pushing off.

**The Solution:** An internal sequence latch (`g_mid_seen`):
1. On power-up or whenever entering `SWING`, `g_mid_seen` is cleared to `0`.
2. When ground contact occurs, the force enters the loading band ($0.2\text{V} \text{ to } 0.6\text{V}$). Because `g_mid_seen == 0`, the state is classified as `LOADING`.
3. When the patient bears full weight, the force enters the mid-stance band ($> 0.8\text{V}$). State becomes `MID-STANCE`, and the latch is set: `g_mid_seen = 1`.
4. As the amputee rolls forward to push off, force decreases back into the $0.2\text{V} \text{ to } 0.6\text{V}$ band. The decision engine evaluates `g_mid_seen`. Because it is `1`, it immediately transitions to `TERMINAL-STANCE`, ensuring correct valve stiffening during push-off.
5. In Phase 5 (`PRE-SWING`), the leg unloads, transitioning cleanly into `SWING`, where `g_mid_seen` is reset to `0`.

### 6.3 Moment Calculation Modes
The firmware supports three selectable moment equations:
1. **Differential Moment (`SM_MOMENT_DIFF` - Default):**
   $$M = (V_{\text{load1}} - V_{\text{load2}}) \times \text{Arm} \quad (\text{Default Arm} = 0.20\,\text{m})$$
2. **Absolute Differential Moment (`SM_MOMENT_ABSDIFF`):**
   $$M = |V_{\text{load1}} - V_{\text{load2}}| \times \text{Arm}$$
3. **Combined Force × Arm (`SM_MOMENT_FORCE_ARM`):**
   $$M = F_{\text{combined}} \times \text{Arm}$$

---

## 7. Motor-Control Slave Link Protocol & Actuator Driving

The STM32U585 acts as the **Master Controller**, commanding a dedicated **Motor Slave Microcontroller** that physically drives the hydraulic spool valve via **USART3 (PD8/PD9)** at 115200 baud.

### 7.1 Master-to-Slave Command Frame (7 Bytes)
On every state transition, the master transmits a 7-byte command frame:
```
[0] 0xA5       : Synchronization Header Byte 1
[1] 0x5A       : Synchronization Header Byte 2
[2] 0x01       : Packet Type (0x01 = State Command)
[3] seq        : Rolling Sequence Counter (uint8)
[4] cmd        : Commanded State (0 = SWING, 1 = STANCE)
[5] crc_lo     : CRC-16/MODBUS Low Byte (covers bytes 0-4)
[6] crc_hi     : CRC-16/MODBUS High Byte
```
* **SWING Command (0):** Hydraulic valve opens to allow low-resistance, free-swinging leg movement.
* **STANCE Command (1):** Hydraulic valve closes/throttles, stiffening the cylinder to prevent knee buckling under body weight.

### 7.2 Slave-to-Master Feedback Frame (7 Bytes)
The slave returns its instantaneous valve position:
```
[0] 0xA5       : Synchronization Header Byte 1
[1] 0x5A       : Synchronization Header Byte 2
[2] 0x02       : Packet Type (0x02 = Valve Position Feedback)
[3] seq        : Acknowledged Sequence Counter
[4] valve_pos  : Spool Valve Position (0 - 100%)
[5] crc_lo     : CRC-16/MODBUS Low Byte
[6] crc_hi     : CRC-16/MODBUS High Byte
```

### 7.3 Noise Filtering & Freshness Gating
* **Median-of-3 Filter:** The firmware buffers the three most recent feedback samples and selects the mathematical median to reject acoustic motor spikes.
* **25 ms Freshness Gate:** If no valid slave frame arrives within $25\,\text{ms}$, the valve position is flagged stale, and a communication warning bit is asserted in the fault register.

---

## 8. Telemetry Protocols: Fixed 25-Byte v1 Frame vs 31-Byte Encrypted v2 Frame

### 8.1 Standard Plaintext Telemetry Frame Layout (v1: 25 Bytes)
Streamed every $10\,\text{ms}$ ($100\,\text{Hz}$) over USART1 at 115200 baud ($2500\,\text{bytes/sec}$ bandwidth requirement, consuming only $21.7\%$ of channel capacity):

| Byte Offset | Field Name | Data Type | Scale / Multiplier | Engineering Range | Description |
|---|---|---|---|---|---|
| **0** | `sync_0` | uint8 | Fixed `0xAA` | - | Frame Start Delimiter Byte 1 |
| **1** | `sync_1` | uint8 | Fixed `0x55` | - | Frame Start Delimiter Byte 2 |
| **2** | `sequence` | uint8 | 1 | 0 to 255 | Rolling sequence counter for packet drop detection |
| **3–4** | `force` | int16 | $\times 100$ | $-327.68 \text{ to } +327.67\,\text{N}$ | Calibrated ground reaction force (Little-Endian) |
| **5–6** | `moment` | int16 | $\times 100$ | $-327.68 \text{ to } +327.67\,\text{Nm}$ | Calibrated sagittal bending moment (Little-Endian) |
| **7–8** | `kneeAngle` | int16 | $\times 100$ | $-327.68 \text{ to } +327.67^\circ$ | Calibrated joint flexion angle (Little-Endian) |
| **9–10** | `valve` | uint16 | 1 | $0 \text{ to } 100\,\%$ | Hydraulic spool valve actuator position |
| **11–12** | `battery` | uint16 | $\times 100$ | $0.00 \text{ to } 655.35\,\text{V}$ | Measured battery terminal voltage ($0-4.2\,\text{V}$ scale) |
| **13** | `state` | uint8 | Enum (0–5) | 0:SW, 1:ST, 2:LO, 3:MS, 4:TS, 5:PS | Active clinical gait state machine phase |
| **14–15** | `fault` | uint16 | Bitmask | 16 individual bits | Active system warning and critical fault flags |
| **16–22** | `reserved` | 7 × uint8| Zero | All zeros | Alignment padding for future expansion |
| **23–24** | `crc16` | uint16 | Polynomial `0x8005` | `0x0000` to `0xFFFF` | CRC-16/MODBUS computed over bytes 0 through 22 |

### 8.2 Encrypted Telemetry Frame Layout (v2: 31 Bytes)
When AES encryption is enabled, bytes 3 through 22 are encrypted:
```
Byte:   [0]   [1]   [2]    [3 ........................... 22]    [23 .................... 30]
Field:  0xAA  0x55  seq    Ciphertext (20 Bytes of Encrypted)    GCM Authentication Tag (8 B)
```
* **Bytes 0–2 (`AA 55 seq`):** Sent in plaintext as Additional Authenticated Data (AAD).
* **Bytes 3–22 (Payload):** 20 bytes of sensor payload encrypted via AES-256-CTR mode.
* **Bytes 23–30 (Tag):** 8 most significant bytes (MSB-8) of the GCM-128 authentication tag.

---

## 9. Cryptographic Security: Hardware AES-256-GCM Implementation

### 9.1 Motivation: Wireless & Sniffable Link Security
When telemetry is transmitted over wireless Bluetooth LE (BLE) or shared clinical networks, patient biometric gait profiles and device calibration coefficients could be intercepted or spoofed. The firmware implements **military-grade AES-256-GCM (Galois/Counter Mode)** providing both **confidentiality** and **tamper-proof cryptographic authentication**.

### 9.2 Key Derivation & Session Setup
```
[Pre-Shared Key (PSK: 32 Bytes)]  +  [Boot Nonce (N: 16 Bytes from HW TRNG)]
                    |
                    v
    Session Key (SK: 32 Bytes) = AES-256-ECB(PSK, N) || AES-256-ECB(PSK, N ^ 0x01)
```
1. **The Pre-Shared Key (PSK):** 256-bit key stored securely in STM32 Flash Bank 2 (Page 126).
2. **Boot Nonce ($N$):** Generated at boot by the on-chip True Random Number Generator (TRNG) peripheral.
3. **Hello Handshake (19 Bytes):** Firmware transmits `A5 5A 10 [16-Byte Nonce]` to the host.
4. **Session Key ($SK$):** Both firmware and Desktop Studio independently derive the identical 256-bit Session Key using the formula above.

### 9.3 Per-Frame Rolling Initialization Vector (IV)
To prevent replay attacks, every frame uses a unique 96-bit IV:
$$\text{IV} = \text{Nonce}[0..8] \;\|\; \text{FrameCounter}_{32\text{-bit Big-Endian}}$$
The frame counter increments on every transmission. Even if an attacker records an identical gait cycle, the ciphertext and authentication tag will be completely different.

### 9.4 Hardware Acceleration & Fail-Safe Software Fallback
* **Hardware AES:** The core 14-round AES-256 block transform executes in the STM32U585 hardware AES peripheral at register level.
* **Software Fallback:** A FIPS-197 compliant software AES implementation is compiled into the binary. At startup, the firmware runs a known-answer self-test. If the hardware engine fails, it seamlessly switches to software fallback, preventing system bricking.

---

## 10. Fault Diagnostics & 16-Bit Safety Register

System health is communicated in every telemetry frame via a **16-bit status bitmask (`status_register_t`)**:

| Bit | Bitmask | Fault Identifier | Trigger Condition | Severity & System Action |
|---|---|---|---|---|
| **0** | `0x0001` | `STATUS_FORCE_LOW` | $F < \text{Threshold}_{\text{min}}$ | Warning: Load cell reading below calibrated floor |
| **1** | `0x0002` | `STATUS_FORCE_HIGH` | $F > \text{Threshold}_{\text{max}}$ | Critical: Excessive ground impact; possible mechanical overload |
| **2** | `0x0004` | `STATUS_MOMENT_LOW` | $M < \text{Threshold}_{\text{min}}$ | Warning: Sagittal bending moment lower limit breached |
| **3** | `0x0008` | `STATUS_MOMENT_HIGH` | $M > \text{Threshold}_{\text{max}}$ | Critical: Severe torque on socket; risk of amputee fall |
| **4** | `0x0010` | `STATUS_KNEE_LOW` | $\theta < \text{Threshold}_{\text{min}}$ | Warning: Hyperextension detection ($< 0^\circ$) |
| **5** | `0x0020` | `STATUS_KNEE_HIGH` | $\theta > \text{Threshold}_{\text{max}}$ | Warning: Excessive flexion ($> 120^\circ$) |
| **6** | `0x0040` | `STATUS_VALVE_LOW` | $\text{Valve} < \text{Threshold}_{\text{min}}$ | Actuator Warning: Spool valve position below commanded setpoint |
| **7** | `0x0080` | `STATUS_VALVE_HIGH` | $\text{Valve} > \text{Threshold}_{\text{max}}$ | Actuator Warning: Spool valve position above commanded setpoint |
| **8** | `0x0100` | `STATUS_BATTERY_LOW` | $V_{\text{bat}} < 3.30\,\text{V}$ | Critical: Battery depleted; alerts user to charge prosthesis |
| **9** | `0x0200` | `STATUS_BATTERY_HIGH` | $V_{\text{bat}} > 4.25\,\text{V}$ | Warning: Overvoltage condition detected on DC rail |
| **10**| `0x0400` | `STATUS_SENSOR1_FAULT` | Stuck 0V, VREF, or Flatline | Force sensor 1 physical disconnection or hardware failure |
| **11**| `0x0800` | `STATUS_SENSOR2_FAULT` | Stuck 0V, VREF, or Flatline | Force sensor 2 physical disconnection or hardware failure |
| **12**| `0x1000` | `STATUS_SENSOR3_FAULT` | Stuck 0V, VREF, or Flatline | Knee angle potentiometer wiper open-circuit |
| **13**| `0x2000` | `STATUS_SENSOR4_FAULT` | Reserved Sensor Error | Reserved auxiliary channel failure |
| **14**| `0x4000` | `STATUS_ADC_STUCK_ZERO`| ADC raw count == 0 for 500 ms | Hardware Fault: ADC converter lockup at ground |
| **15**| `0x8000` | `STATUS_ADC_STUCK_VREF`| ADC raw count == 4095 for 500 ms| Hardware Fault: ADC converter lockup at reference voltage |

---

## 11. In-Situ Calibration, Thresholds & Internal Flash Bank 2 Persistence

### 11.1 Threshold Packet (`0x31` — 45 Bytes)
Transmitted by the Desktop Studio to define dynamic operating boundaries:
* **Header:** `0xA5`, `0x31` (2 Bytes)
* **Payload:** 10 × 32-bit IEEE-754 single-precision floating-point values ($40\,\text{bytes}$):
  * `force_min`, `force_max`
  * `moment_min`, `moment_max`
  * `knee_min`, `knee_max`
  * `valve_min`, `valve_max`
  * `battery_min`, `battery_max`
* **Footer:** CRC-16/MODBUS ($2\,\text{bytes}$)
* **Action:** Firmware validates CRC and immediately applies thresholds to the live fault detection engine.

### 11.2 Coefficients Packet (`0x32` — 53 Bytes)
Allows field calibration of sensor gain and baseline offsets:
* **Header:** `0xA5`, `0x32` (2 Bytes)
* **Payload:** 12 × 32-bit single-precision floats ($48\,\text{bytes}$):
  * Scale factor, zero offset, and reference voltage for all sensor channels.
* **Footer:** CRC-16/MODBUS ($2\,\text{bytes}$)

### 11.3 Protected Flash Persistence Workflow
To prevent flash memory corruption during active 100 Hz streaming, the Desktop Studio and firmware execute a **strict safety interlocking sequence**:
```
Clinician enters Admin Password in Desktop Studio
                     |
                     v
Desktop Studio sends: "halt on" CLI command
                     |
                     v
STM32 Firmware halts 100 Hz Telemetry Timer (Safe Non-Functional State)
                     |
                     v
Desktop Studio transmits 53-byte 0x32 Coefficient Packet
                     |
                     v
Firmware verifies CRC-16 -> Unlocks Flash Bank 2 -> Erases Page -> Programs Quadwords -> Verifies Magic
                     |
                     v
Desktop Studio sends: "halt off" CLI command
                     |
                     v
STM32 Firmware resumes 100 Hz Telemetry with new persistent calibration
```
* **Boot Reload:** During initial power-on in `main.c`, the firmware inspects Flash Bank 2. If valid magic words and CRC match, calibrated coefficients are restored automatically.

---

## 12. Desktop Studio Architecture (Electron Multi-Process Model)

Prosthetic Telemetry Studio v1.3.0 is built on **Electron 31**, structured with strict process isolation to ensure medical software reliability:

```
+---------------------------------------------------------------------------------------------------+
|                                 MAIN PROCESS (Node.js Environment)                                |
|  • Native Window Management & Menu Bars                                                           |
|  • Native Serial Communications (node-serialport @ 115200 Baud)                                   |
|  • Direct File System Access: Native CSV / JSON Session Export Dialogs                            |
|  • Chromium Permission Interception & Single-Instance Application Locking                         |
+-------------------------------------------------+-------------------------------------------------+
                                                  |
                                                  | contextBridge IPC (Strict Security Boundary)
                                                  v
+-------------------------------------------------+-------------------------------------------------+
|                                PRELOAD SCRIPT (preload.js)                                        |
|  • Context Isolation: ENABLED | Node Integration: DISABLED | Remote Module: DISABLED             |
|  • Exposes safe, immutable window.electronAPI: sendCommand(), onData(), exportFile()              |
+-------------------------------------------------+-------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
|                              RENDERER PROCESS (Chromium Front-End)                                |
|  • HTML5 / Glassmorphism CSS UI Engine                                                            |
|  • 100 Hz Frame Parser: Sync Search (0xAA 0x55) -> Extraction -> CRC Check -> Field Unpacking     |
|  • Plotly.js 2.35.2 WebGL Multi-Pane Charting Engine                                              |
|  • High-Speed HTML5 Canvas 2D Glow Oscilloscope                                                   |
|  • Interactive Three.js / WebGL 3D State Space Visualizer                                         |
|  • Automated Gait Analytics Processor (Cadence, Symmetry Index, Stride Segmentation)              |
|  • Transports: Native USB SerialPort, Web Serial API Fallback, Web Bluetooth NUS, 100 Hz Demo    |
+---------------------------------------------------------------------------------------------------+
```

---

## 13. Detailed Guide to All 16 Desktop Studio GUI Provisions

Every tab in Prosthetic Telemetry Studio v1.3.0 was designed for specific clinical and engineering workflows:

### Tab 1: Overview Dashboard (`01_overview.png`)
* **Clinical Purpose:** Primary monitoring view for clinicians during patient walking trials.
* **Key Provisions:**
  * **11 Live KPI Cards:** Large digital readouts displaying Instantaneous Force (N), Moment (Nm), Knee Angle (°), Spool Valve Position (%), Battery Voltage (V), Gait Phase Badge (color-coded), and Fault Status.
  * **Packet Rate Monitor:** Digital indicator verifying consistent 100 Hz reception ($100 \pm 1\,\text{packets/sec}$) with latency monitoring in milliseconds.
  * **CRC Integrity Meter:** Real-time health bar indicating $100\%$ valid packet checksums.
  * **Quick Action Controls:** One-click buttons to Start/Stop Session Recording, Pause Stream, and trigger synthetic Demo Mode.
  * **Stale Data Alarm:** Automatically displays a prominent warning banner if no packet arrives for $> 1.0\,\text{second}$.

### Tab 2: Real-Time Multi-Pane Waveforms (`02_waveforms.png`)
* **Clinical Purpose:** Dynamic physiological time-series visualization.
* **Key Provisions:**
  * **Four Synchronized Panes:**
    1. *Vertical Force (N):* Shows heel-strike peaks and push-off unloading.
    2. *Sagittal Moment (Nm):* Visualizes joint flexion/extension torque.
    3. *Knee Joint Angle (°):* Displays physical flexion throughout the stride.
    4. *Dual-Axis Valve (%) & Battery (V):* Tracks actuator modulation and battery drain.
  * **Time-Window Selectors:** Toggle between 5s, 10s, 30s, and 60s historical views.
  * **20 Hz Batch Flush Architecture:** Raw 100 Hz packets are queued in memory and flushed to Plotly.js at 20 Hz, preventing UI thread lockup.

### Tab 3: PedoGraph-Style Hardware Oscilloscope (`03_scope.png`)
* **Clinical Purpose:** High-speed raw signal inspection mimicking laboratory oscilloscopes.
* **Key Provisions:**
  * **Phosphor Glow Traces:** Rendered directly to HTML5 Canvas using additive blending and double-stroke rendering for an authentic CRT phosphor appearance.
  * **DPR-Sharp Scaling:** Automatically scales with the monitor's Device Pixel Ratio, ensuring ultra-crisp traces on 4K and Retina displays.
  * **Interactive Channel Toggle Chips:** Clinicians can enable or isolate individual signals (Force, Moment, Knee Angle, Valve, Battery) with a single click.
  * **Independent Channel Autoscale:** Automatically optimizes the vertical gain for each channel independently.

### Tab 4: 3D State Space Trajectory (`04_3d_view.png`)
* **Clinical Purpose:** Advanced biomechanical phase-space analysis.
* **Key Provisions:**
  * **Tri-Axial WebGL Phase Space:** Plots Knee Angle ($X$) vs Force ($Y$) vs Moment ($Z$) in a fully interactive 3D volume.
  * **Gait Phase Color Coding:** Trail points are dynamically colored by gait state (Green = Swing, Orange = Loading, Red = Mid-Stance, Purple = Terminal-Stance).
  * **Gait Stability Loops:** Normal human walking generates closed, repeatable orbital loops in state space. Stumbling, fatigue, or joint misalignment causes immediate orbit distortion.
  * **Camera Controls:** Supports mouse drag rotation, pinch zoom, panning, and instant camera reset.

### Tab 5: Automated Clinical Gait Analytics (`05_analytics.png`)
* **Clinical Purpose:** Quantitative gait assessment metrics for prosthetists and physical therapists.
* **Key Provisions:**
  * **Cadence & Stride Time:** Automated computation of steps per minute and individual stride durations (with $0.3\text{s} \text{ to } 3.5\text{s}$ validity gating).
  * **Stance / Swing Ratio:** Displays exact physiological percentages (e.g., $62\%$ Stance / $38\%$ Swing).
  * **Peak Impact & Knee ROM:** Tracks maximum ground reaction force (N) and joint Range of Motion (ROM in degrees).
  * **Symmetry Index & Histogram:** Quantitative symmetry index comparing step duration variability, accompanied by a stride-time distribution histogram.

### Tab 6: Protocol & Frame Inspector (`12_inspector.png`)
* **Clinical Purpose:** Low-level engineering debugging of the binary telemetry stream.
* **Key Provisions:**
  * **Color-Coded Hex Grid:** Displays the raw 25-byte binary frame with distinct color highlighting for Sync Bytes (Blue), Sequence (Cyan), Sensors (Green), Gait State (Orange), Fault Bitmask (Red), and CRC-16 (Purple).
  * **Decoded Fields Table:** Full breakdown of each field showing Byte Offset, Type, Raw Hex, Scale Factor, and Final Engineering Value.
  * **Live CRC Verifier:** Displays received CRC vs calculated CRC (e.g., `Received: 0xD7CB | Calculated: 0xD7CB -> MATCH`).
  * **Diagnostic Parser Stream:** Timestamped logging of parser synchronization events, frame drops, and re-sync recovery.

### Tab 7: Safety Thresholds Configuration (`10_thresholds.png`)
* **Clinical Purpose:** In-situ adjustment of patient safety operating boundaries.
* **Key Provisions:**
  * **Dynamic Sliders & Input Boxes:** Min and Max thresholds for Force, Moment, Knee Angle, Valve, and Battery.
  * **Transmit Threshold Packet (0x31):** Packages settings into a 45-byte binary packet and transmits to MCU flash.
  * **Live Violation Highlighting:** Threshold fields glow red if the current live sensor reading violates the parameter.

### Tab 8: Sensor Calibration & Coefficients (`11_coefficients.png`)
* **Clinical Purpose:** Sensor gain and zero-offset calibration without recompiling firmware.
* **Key Provisions:**
  * **Scale, Offset & Reference Voltages:** Configurable multipliers for all 3 sensor channels.
  * **Password-Protected Flash Write:** Requires entering an administrator password, automatically halting 100 Hz timers, writing Flash Bank 2, and resuming streaming.
  * **Calibration Check Tool:** Live zero-baseline verification wizard.

### Tab 9: Clinical Session Recording (`07_recording.png`)
* **Clinical Purpose:** Continuous session logging for clinical research and patient records.
* **Key Provisions:**
  * **Session Control Bar:** Start, Pause, and Stop recording controls with real-time sample counters.
  * **Native Export Dialogs:** One-click export of complete 100 Hz synchronized datasets into standard CSV (Excel/MATLAB) or JSON formats.
  * **Session History:** Historical browser showing duration, total sample count, and recorded file sizes.

### Tab 10: Patient & Prosthetic Metadata (`06_participant.png`)
* **Clinical Purpose:** HIPAA-compliant demographic tracking linked to recorded sessions.
* **Key Provisions:**
  * **Patient Fields:** Subject ID, Amputation Level (Transfemoral/Knee Disarticulation), Affected Side (Left/Right).
  * **Prosthetic Hardware Profile:** Prosthetic Socket Type, Knee Mechanism Type, Foot Module, and Clinician Name.
  * **Local Storage Only:** All patient identifiers remain encrypted on the local workstation with zero cloud transmission.

### Tab 11: Live Firmware Configuration (`15_fwconfig.png`)
* **Clinical Purpose:** Real-time state machine parameter tuning without firmware flashing.
* **Key Provisions:**
  * **Gait Timing Sliders:** Adjust window size ($1 \text{ to } 255 \text{ cycles}$) and moment arm distance ($0.05 \text{ to } 0.50\,\text{m}$).
  * **Combination Mode Selector:** Toggle between SUM, AVERAGE, MAX, and MIN.
  * **Phase Band Adjusters:** Real-time adjustment of voltage boundaries for Swing, Pre-Swing, Loading, Mid-Stance, and Stance.
  * **Automatic CLI Translation:** Automatically formats and transmits corresponding CLI strings (`set cycles 10`, `set arm 0.20`, etc.).

### Tab 12: Interactive Serial Monitor (`13_serialmon.png`)
* **Clinical Purpose:** Low-level developer command-line terminal.
* **Key Provisions:**
  * **Dual ASCII / Hex View:** Displays raw bidirectional UART traffic.
  * **Quick-Command Action Buttons:** Pre-configured macro buttons for instant execution of common commands: `status`, `sensors`, `demo`, `mode real`, `halt on`, `halt off`, `reboot`.
  * **Terminal Controls:** Auto-scroll lock, clear screen, and command history buffer.

### Tab 13: Custom Biomechanical XY Plotting (`14_xyplot.png`)
* **Clinical Purpose:** Flexible cross-variable trajectory plotting.
* **Key Provisions:**
  * **Arbitrary Axis Mapping:** Plot any telemetry variable against any other (e.g., Force vs Knee Angle, Moment vs Force, Valve vs Knee Angle).
  * **Hysteresis Visualization:** Directly visualizes biomechanical hysteresis loops during walking strides.
  * **Persistence Buffer:** Configurable trailing sample count ($100 \text{ to } 2000\,\text{points}$).

### Tab 14: Connectivity & Hardware Selection (`09_connection.png`)
* **Clinical Purpose:** Managing physical and wireless transport layers.
* **Key Provisions:**
  * **USB SerialPort Selector:** Detects available COM ports with baud rate configuration (Default: 115200).
  * **Web Bluetooth (BLE) Manager:** Scans for devices advertising the Nordic UART Service (`SmartKnee-U585`).
  * **100 Hz Synthetic Gait Demo Engine:** Built-in biomechanical simulation engine allowing complete software demonstration without physical hardware.

### Tab 15: Communication Protocol Reference Tab (`08_protocol.png`)
* **Clinical Purpose:** Onboard developer documentation.
* **Key Provisions:**
  * **Interactive Byte Map:** Visual, clickable documentation of the 25-byte v1 and 31-byte v2 encrypted frame formats.
  * **Bitmask Cheat Sheet:** Complete bitwise explanation of all 16 fault register flags.

### Tab 16: System Information & Architecture Tab (`16_about.png`)
* **Clinical Purpose:** System versioning and architecture overview.
* **Key Provisions:**
  * **Version Badge:** Displays application version (v1.3.0), Electron version (31.6.0), Node version (24.x), and build date.
  * **System Topology Diagram:** High-level schematic showing hardware and software interactions.

---

## 14. Engineering Findings, Discrepancies & Root-Cause Fixes

A thorough audit of the firmware codebase against legacy documentation identified several critical discrepancies that have been resolved in the codebase:

### Discrepancy 1: Motor Slave Link UART Assignment
* **Legacy Documentation:** Stated that the motor slave communicates over `USART2` on pins `PD5 (TX)` and `PD6 (RX)`.
* **Authoritative Code (`usart.c`, `slave_comm.c`):** The functional and hardware-verified implementation strictly uses **`USART3`** on pins **`PD8 (TX)`** and **`PD9 (RX)`**.
* **Impact & Resolution:** Documentation has been corrected to prevent hardware wiring errors during cable harness assembly.

### Discrepancy 2: ADC Clock Frequency
* **Legacy Changelogs:** Mentioned an ADC conversion clock of $16\,\text{MHz}$.
* **Authoritative Code (`adc.c`):** The ADC is clocked from the **$4\,\text{MHz}$ MSI clock (MSIC)**.
* **Engineering Rationale:** At $4\,\text{MHz}$, conversion noise is significantly lower, maximizing Effective Number of Bits (ENOB) and preventing thermal drift on high-impedance strain gauges.

### Discrepancy 3: Fault Status Register Width
* **Legacy Application Drafts:** Parsed the fault status as an 8-bit unsigned integer (`uint8`).
* **Authoritative Implementation (`smart_protocol.c`, `constants.js`):** The fault register is an **expanded 16-bit bitmask (`uint16`) located at Byte offset 14**. Studio v1.3.0 correctly unpacks both bytes.

### Discrepancy 4: Battery Voltage Scale Mismatch
* **Default Studio Config:** Studio's default threshold presets were historically set for a 12V lead-acid battery ($10.5\text{V} \text{ to } 13.0\text{V}$).
* **Firmware Implementation:** The prosthetic knee uses a single-cell lithium-ion battery with a resistor divider, producing a **$0.0\text{V} \text{ to } 4.2\text{V}$ physical scale**.
* **Action Required:** Update `constants.js` default preset limits to $\text{Min} = 3.3\,\text{V}$ and $\text{Max} = 4.2\,\text{V}$.

---

## 15. Verification Status Matrix & Bench Test Results

| Subsystem / Feature | Module | Implementation State | Verification Method & Results |
|---|---|---|---|
| **ADC1 Scan & DMA Pipeline** | `adc.c`, `adc_manager.c` | Complete | **Hardware Verified:** Oscilloscope and live strain gauge testing verified continuous 3-channel sampling with $0\%$ CPU load. |
| **51-Tap FIR Low-Pass Filter** | `fir_filter.c` | Complete | **Bench Verified:** Tested with simulated impulse noise; confirmed $> 40\,\text{dB}$ attenuation at $25\,\text{Hz}$ cutoff. |
| **100 Hz Telemetry Stream** | `smart_protocol.c` | Complete | **Hardware Verified:** Captured via logic analyzer and PC COM port; verified stable 10 ms interval ($\pm 0.05\,\text{ms}$ jitter). |
| **Gait State Machine** | `state_machine.c` | Complete | **Simulated & Bench Verified:** Synthetic and manual sensor inputs confirmed proper sequencing and `mid_seen` latching. |
| **USART3 Motor Slave Link** | `slave_comm.c` | Complete | **Hardware Verified (Round 2):** Bi-directional communication with motor slave confirmed with median filtering. |
| **16-Bit Fault Bitmask Engine** | `thresholds.c`, `sensor.c`| Complete | **Firmware Verified:** Induced open-circuit and short-circuit faults verified instant assertion of corresponding status bits. |
| **Calibration & Flash Bank 2** | `coefficients.c`, `crypto.c`| Complete | **Bench Verified:** Write/read endurance verified across reboot cycles; magic word and CRC validation operating. |
| **Hardware AES-256-GCM** | `crypto.c` | Complete | **Host Vectors Pass:** Verified bit-for-bit against OpenSSL 3.0 and Python `pycryptodome` test vectors. Physical bench pending. |
| **Desktop Studio Transports** | `main.js`, `preload.js` | Complete | **Functionally Verified:** Tested with native USB Virtual COM Port, Web Serial, and 100 Hz Demo Mode. |
| **Studio UI (All 16 Tabs)** | `index.html`, `renderer.js` | Complete | **Fully Captured & Reviewed:** All 16 views running live with dynamic chart updates, 3D WebGL rendering, and recording. |
| **Session Recording & Export** | `renderer.js` | Complete | **Functionally Verified:** Multi-minute continuous sessions exported to CSV and JSON; validated in Excel and Python. |

---

## 16. Strategic Roadmap & Recommended Next Steps

### Step 1: Reconcile Documentation & Presets (Immediate)
* Update `constants.js` in the Desktop Studio to set battery warning thresholds to $3.3\,\text{V} \text{ (Low)}$ and $4.2\,\text{V} \text{ (High)}$.
* Update all wiring schematics to definitively specify `USART3` on `PD8/PD9` for the motor slave.

### Step 2: Implement Automated Host-Based Test Suite
* Develop a Python-based CI test script that feeds raw byte streams into the protocol parser to verify:
  * CRC-16/MODBUS calculation across boundary conditions.
  * Sequence counter rollover handling ($255 \rightarrow 0$).
  * Frame resynchronization after corrupted byte insertion.

### Step 3: Implement Automated Headless Firmware CI
* Set up a GitHub Actions workflow or local build script using `arm-none-eabi-gcc` to:
  * Build the STM32CubeIDE project in headless mode on every commit.
  * Run static analysis (cppcheck) to verify zero buffer overflows in DMA buffers.

### Step 4: Finalize End-to-End Cryptographic Bench Acceptance
* Execute a physical hardware test session where the STM32U585 board streams encrypted v2 frames over USB/BLE to Studio v1.3.0 for $> 1\,\text{hour}$ without packet drop or re-keying failure. Log metrics in `docs/CRYPTO_BENCH.md`.

### Step 5: Clinical Gait Trials & Human-Factors Validation
* Deploy Studio v1.3.0 on clinician surface tablets during treadmill walking trials with transfemoral amputees to evaluate:
  * Visual legibility of the PedoGraph oscilloscope under hospital lighting.
  * Responsiveness of the automated symmetry index metric during cadence changes.

---

## 17. Complete Technical Glossary

* **ADC (Analog-to-Digital Converter):** An on-chip peripheral that measures continuous physical analog voltages and converts them into digital numbers ($12\text{-bit} = 0 \text{ to } 4095$).
* **GPDMA (General-Purpose Direct Memory Access):** A hardware subsystem in the STM32U5 that transfers data directly between peripherals (ADC/UART) and memory without involving the Cortex-M33 CPU.
* **FIR Filter (Finite Impulse Response Filter):** A digital signal processing filter whose response to an impulse settles to zero in finite time; guaranteed to be inherently stable with linear phase response.
* **CRC-16/MODBUS (Cyclic Redundancy Check):** An error-detecting code utilizing the polynomial $x^{16} + x^{15} + x^2 + 1$ (`0x8005`, initial value `0xFFFF`) to verify that telemetry packets have not suffered electrical bit corruption.
* **FreeRTOS:** A market-leading, open-source real-time operating system kernel designed for microcontrollers, providing preemptive scheduling, software timers, and inter-task communication.
* **AES-256-GCM (Advanced Encryption Standard in Galois/Counter Mode):** A symmetric authenticated cipher providing both confidentiality (AES-CTR) and data integrity verification (GHASH authenticator).
* **PSK (Pre-Shared Key):** A secret 256-bit cryptographic key known beforehand to both the microcontroller and the authorized Desktop Studio workstation.
* **TRNG (True Random Number Generator):** A hardware peripheral that generates unpredictable entropy from physical thermal noise inside the silicon chip.
* **Spool Valve:** A precision hydraulic valve comprising an internal cylindrical spool that slides to throttle or bypass fluid, adjusting the mechanical damping of the prosthetic knee.
* **Gait Cycle:** The repetitive sequence of foot movements during walking, beginning with heel strike of one foot and ending with the subsequent heel strike of the same foot.
* **Cadence:** The rate of walking, expressed quantitatively as the number of steps taken per minute ($\text{steps/min}$).
* **ROM (Range of Motion):** The angular displacement of a biological or prosthetic joint through its full flexion and extension arc (measured in degrees).
* **Electron:** An open-source desktop application framework combining the Chromium web rendering engine and the Node.js runtime environment.
* **contextBridge:** An Electron security API that safely exposes selected inter-process communication (IPC) methods from a privileged preload script to an untrusted renderer page.
