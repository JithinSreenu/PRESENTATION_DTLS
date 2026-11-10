# Prosthetic Knee Telemetry System: Technical Deep-Dive & Architecture Explanations

> **Author:** Jithin Sreenu  
> **Target:** STMicroelectronics B-U585I-IOT02A (STM32U585) & Prosthetic Telemetry Studio (Electron)  
> **Purpose:** Quick-Access Companion Guide breaking down every specific engineering decision, register fix, and software boundary.

---

## Core Formula Breakdown:
$$\text{\textbf{ADC1 + GPDMA1, 3 channels, 32 samples/ch, 51-tap FIR @ 25 Hz}}$$

### 1. What does each piece actually mean?

| Term | What it is in simple terms | What it does in your prosthetic knee |
|---|---|---|
| **ADC1** | Analog-to-Digital Converter 1 | Measures continuous analog voltage ($0\text{ to }3.3\text{V}$) on the pins and converts it into digital numbers ($0\text{ to }4095$, 12-bit). |
| **GPDMA1** | General-Purpose Direct Memory Access | A dedicated hardware co-processor inside the STM32U5. It moves the ADC numbers **directly into RAM automatically**, with **$0\%$ CPU usage** (the main Cortex-M33 processor does not waste a single clock cycle copying data). |
| **3 channels** | 3 physical sensor pins | • **PC0 (A0):** Load Cell 1 (Ground Reaction Force)<br>• **PC2 (A1):** Load Cell 2 (Sagittal Moment / Force Balance)<br>• **PC4 (A2):** Knee Joint Angle Potentiometer |
| **32 samples/ch** | Multi-sample buffer size | For each of the 3 channels, the ADC takes **32 consecutive readings** before giving a result. That is $3 \times 32 = 96$ measurements in total per block. |
| **51-tap FIR @ 25 Hz** | Digital Noise Filter | A Finite Impulse Response (FIR) low-pass filter with 51 mathematical coefficients that **cuts off any vibration or noise above 25 Hz**, while preserving smooth biological walking curves. |

---

### 2. Why specifically 32 samples per channel?

Why not take just 1 sample? Why not 10? Why 32?

There are **4 critical engineering reasons**:

#### ① Noise Reduction & Gaining Extra Resolution (Oversampling)
When an amputee's prosthetic foot strikes the ground, the load cell doesn't produce a calm, clean voltage. It experiences:
* High-frequency acoustic ringing (metallic vibrations of the titanium tube and foot shell)
* Electrical spikes from nearby motor switching (PWM noise)

If you take only **1 snapshot**, you might sample right on a spike or dip, giving a false reading (e.g., reading $600\,\text{N}$ when the true load is only $400\,\text{N}$).

By taking **32 samples** in a rapid burst and averaging them, random electrical and acoustic noise cancels itself out:
$$\text{Extra Resolution Bits} = \frac{1}{2} \log_2(N) = \frac{1}{2} \log_2(32) = 2.5 \text{ bits}$$
This means your **12-bit ADC acts with the precision of a 14.5-bit ADC**! Small force changes of less than $1\,\text{N}$ become clearly visible.

---

#### ② Power of 2 ($2^5 = 32$) = Instant Microcontroller Math
On an embedded microcontroller, normal division (like dividing a sum by 30 or 33) requires an expensive hardware division instruction that takes multiple CPU clock cycles.

Because **32 is $2^5$**, division is done in **1 single CPU clock cycle** using a bitwise right-shift:
$$\text{Average} = \text{Sum} \gg 5$$
This allows the firmware to compute the average of 32 samples in a fraction of a microsecond.

---

#### ③ Perfect Timing Math ($360\,\mu\text{s}$ vs $10\,\text{ms}$)
Look at how fast the STM32U5 ADC operates with its 4 MHz clock:
* **1 channel conversion:** 15 clock cycles = $3.75\,\mu\text{s}$
* **1 full scan (3 channels):** 45 clock cycles = $11.25\,\mu\text{s}$
* **32 scans:** $32 \times 45 = 1,440 \text{ clock cycles} = \mathbf{360\,\mu s} \text{ (0.36 ms)}$

```
+-------------------------------------------------------------+
|               One 10 ms Telemetry Period (100 Hz)           |
+-------------------------------------------------------------+
| 0.36 ms |                    9.64 ms                        |
| 32 scans|   Plenty of quiet time for FIR filter, gait       |
| buffer  |   state machine, and UART DMA transmission        |
+---------+---------------------------------------------------+
```
Because the 32 samples take only **$0.36\,\text{ms}$**, they are completed well before the **$10\,\text{ms}$** ($100\,\text{Hz}$) telemetry tick. Data is always fresh and never delayed.

---

#### ④ Tiny Memory Footprint (192 Bytes)
$$3 \text{ channels} \times 32 \text{ samples} = 96 \text{ halfwords} = \mathbf{192 \text{ bytes}}$$
A 192-byte buffer is microscopic compared to the STM32U585's **768,000 bytes (768 KB) of SRAM**. It leaves virtually all memory free for FreeRTOS tasks, communication queues, and cryptographic sessions.

---

### 3. Why a 51-Tap FIR Filter at 25 Hz?

#### Why 25 Hz?
* **Human walking occurs between 0.5 Hz and 5 Hz.** Even an Olympic sprinter rarely exceeds 8–10 Hz in joint angular velocity.
* Any signal oscillating faster than **25 Hz is pure mechanical impact noise, shoe slap, or motor hum**.
* Setting the filter cutoff frequency to **25 Hz** lets **100% of the patient's biological walking curve pass through untouched**, while cutting away the mechanical impact shock.

#### Why a 51-tap FIR?
* **Zero Phase Distortion (Linear Phase):** Unlike IIR filters (Butterworth, Chebyshev), an FIR filter delays all frequencies by the exact same amount of time. The true shape of the heel-strike curve is preserved without warping.
* **Sharp Filter Rolloff:** 51 coefficients provide $> 40\,\text{dB}$ of stopband attenuation, completely silencing high-frequency noise.

---

# Detailed Point-by-Point Explanations

---

## Part 1: ADC Sampling & The DMA Engine

### 1. `Scan: 3 channels interleaved, 2.5-cycle sampling, 15 ADC cycles/channel`
* **What it means:**  
  The ADC measures 3 pins in a round-robin sequence: **PC0 (Force) $\rightarrow$ PC2 (Moment) $\rightarrow$ PC4 (Knee Angle)**.  
  * *Sampling time:* The internal capacitor samples the pin for $2.5 \text{ clock cycles}$.
  * *Conversion time:* The 12-bit successive-approximation logic takes $12.5 \text{ cycles}$.  
  * *Total time:* $2.5 + 12.5 = \mathbf{15 \text{ clock cycles per channel}}$.
* **Why it matters:**  
  At a 4 MHz clock, 15 cycles takes exactly **$3.75\,\mu\text{s}$**. All 3 channels are scanned in **$11.25\,\mu\text{s}$**. This provides high-speed conversion with zero thermal noise drift on the strain gauges.

---

### 2. `DMA: CFGR1 = CONT | DMNGT_1 — request after each conversion (critical fix)`
* **The Problem It Solved:**  
  During initial hardware testing, all physical sensors read $0$ while only battery voltage updated.
* **Why It Happened:**  
  In the new STM32U5 architecture, the ADC has a Data Management register (`DMNGT`). By default, the ADC only tells DMA to transfer when the *entire sequence* completes, causing data registers to overwrite and drop conversions.
* **The Fix:**  
  Setting `DMNGT = 10b` (`DMNGT_1`) commands the ADC hardware: *"The moment you finish converting ONE channel, immediately request DMA to move it to RAM before starting the next channel."* This completely fixed the zero-data bug.

---

### 3. `Re-arm: inside the DMA-complete ISR (removes Abort+Start race)`
* **What it means:**  
  DMA circular transfers must be re-armed to receive the next batch. In naive firmware, the CPU tries to stop DMA, reconfigure it, and restart it from a normal thread.
* **The Race Condition:**  
  If the timer ticks and tries to restart DMA at the exact millisecond DMA is already transferring, the peripheral hardware crashes into an `ALREADY_RUNNING` error state.
* **The Solution:**  
  Re-arming is done **inside the hardware Interrupt Service Routine (ISR)** automatically when the transfer finishes. This eliminates the race condition because hardware itself signals the restart cleanly.

---

### 4. `Guard: counter-bounded init; failure -> stuck fault, never a hang`
* **What it means:**  
  During boot, the ADC internal voltage regulator must settle, and hardware calibration must run. Typically, engineers write `while(!ADC_Ready);`.
* **The Danger:**  
  If a wire is loose or power dips, that `while` loop runs forever. The entire prosthetic knee **freezes at boot**, and the patient is left with a locked or limp leg.
* **The Guard:**  
  The loop is counter-bounded: `for (int i = 0; i < 1600000; i++)`. If the ADC doesn't respond after 1,600,000 counts, it breaks out, asserts the **`STATUS_ADC_STUCK_ZERO`** fault flag, and lets the RTOS boot normally so alarms can sound and the user is warned.

---

## Part 2: Firmware Hardening & Boot Reliability

### 5. `Boot HardFault: GPDMA1 IRQ enables moved after handle init; stale flags + NVIC pending cleared (SetPriority -> ClearPending -> EnableIRQ)`
* **The Fatal Bug:**  
  On early firmware, turning on the power caused an immediate **HardFault crash** (the MCU red-screened before main loop ran).
* **Root Cause:**  
  The interrupt controller (NVIC) was enabled *before* the STM32 software handles were initialized. The moment interrupts turned on, an interrupt fired into an uninitialized NULL memory pointer.
* **The Fix:**  
  1. Initialize all software handles first.
  2. Set IRQ Priority.
  3. Clear any residual boot noise (`NVIC_ClearPendingIRQ`).
  4. Only then enable the interrupt (`NVIC_EnableIRQ`).

---

### 6. `Interrupt firewall removed: the __disable_irq() band-aid risked leaving PRIMASK set on failure paths`
* **What it means:**  
  Some developers hide boot glitches by calling `__disable_irq()` (shutting off all interrupts globally) around sensitive code.
* **The Danger:**  
  If an error occurs or a function returns early while interrupts are disabled, the `PRIMASK` CPU register stays locked at 1. The FreeRTOS scheduler can never tick, and all timers die permanently.
* **The Fix:**  
  The `__disable_irq()` band-aid was ripped out. Instead, peripheral registers were properly initialized in correct hardware sequence, allowing safe operation without globally killing interrupts.

---

### 7. `TX double-buffer: private dma_tx_buf[256] prevents zero-copy buffer reuse corruption`
* **The Bug:**  
  UART DMA takes $2.17\,\text{ms}$ to transmit 25 bytes. If the telemetry task builds a new packet in the same buffer before the previous DMA transmission has finished shifting out the last byte, the serial stream sends half of the old packet and half of the new packet (corrupting the CRC).
* **The Solution:**  
  The firmware creates a dedicated staging buffer (`dma_tx_buf[256]`). The newly built frame is copied into this dedicated buffer, so the active transmission is never overwritten by subsequent calculations.

---

### 8. `Transport: GPDMA1 Ch0 TX DMA; a frame is dropped on HAL_BUSY (no cadence stall)`
* **What it means:**  
  Telemetry streams at 100 Hz (every 10 ms). If for some reason the previous transmission is still finishing (`HAL_BUSY`), the firmware **drops that single frame** instead of waiting.
* **Why this is critical for a prosthetic limb:**  
  If the firmware waited for the UART to be ready, the entire FreeRTOS scheduler would stall. Damping decisions for the patient would be delayed by milliseconds, causing knee instability. **Dropping a telemetry display packet is harmless; delaying knee control is dangerous.**

---

## Part 3: Clinical Gait State Machine & Motor Control

### 9. `Method: force-band + sequence based (LOADING vs TERMINAL-STANCE split by a mid-stance-seen flag)`
* **The Biomechanical Paradox:**  
  When an amputee steps down (Heel Strike / Loading), ground force is $\approx 400\,\text{N}$. When they push off the toe (Terminal-Stance), ground force is also $\approx 400\,\text{N}$. How can the knee tell whether the user is landing or leaving the ground?
* **The `mid_seen` Solution:**  
  * At Heel Strike $\rightarrow$ State is **LOADING** (because `mid_seen = 0`).
  * When body weight passes directly over the leg $\rightarrow$ Force peaks ($>800\,\text{N}$) $\rightarrow$ State becomes **MID-STANCE**, and the firmware sets `mid_seen = 1`.
  * As the patient pushes off and force falls back to $400\,\text{N}$ $\rightarrow$ The firmware checks `mid_seen`. Because it is `1`, it transitions to **TERMINAL-STANCE**, not Loading!
  * When the leg lifts into the air $\rightarrow$ State becomes **SWING**, and `mid_seen` resets to `0`.

---

### 10. `Slave command: on each transition the master sends SWING/STANCE to the slave`
* **What it means:**  
  The STM32U585 does not directly drive the hydraulic motor coils. Instead, it sends high-level commands over **USART3** to a dedicated slave motor board:
  * **Command 0 (SWING):** Tells the slave: *"Leg is in the air—open the hydraulic valve to 100% so the knee can bend freely."*
  * **Command 1 (STANCE):** Tells the slave: *"Foot is on the ground—throttle the valve to provide hydraulic stiffness and prevent knee collapse."*

---

## Part 4: Safety Diagnostics & Fault Bitmasks

### 11. `Fault register: 16-bit bitmask streamed in every frame (bits 0-13 limits/sensor-fail, 14-15 ADC stuck-zero/VREF)`
* **What it means:**  
  Instead of sending a single number for errors, every frame contains a 16-bit bitmask (`uint16`) where **each bit represents a specific medical/hardware safety check**:
  * *Bits 0–9:* Safety threshold violations (e.g., force too high, knee angle past limits).
  * *Bits 10–13:* Physical sensor disconnections (broken wire or unplugged connector).
  * *Bits 14–15:* ADC converter hardware lockups (stuck at 0V ground or 3.3V power).
* Multiple faults can be reported simultaneously without ambiguity.

---

### 12. `Per-channel sensor check: stuck-zero (~0.5 s), stuck-VREF (~0.5 s), flatline variance (~1.5 s); auto-clears`
* **The 3 Smart Sensor Tests:**
  1. **Stuck-Zero (50 cycles = 0.5s):** If a sensor reads $0\,\text{V}$ continuously, a ground short or wire cut has occurred.
  2. **Stuck-VREF (50 cycles = 0.5s):** If a sensor reads maximum $3.3\,\text{V}$ continuously, a power short has occurred.
  3. **Flatline Variance (150 cycles = 1.5s):** When a human walks, analog signals naturally fluctuate. If the mathematical variance is near zero while walking, the sensor potentiometer wiper has seized mechanically.
* **Auto-Clearing:** If the wire reconnects or noise resumes, the firmware automatically clears the fault bit and restores normal green status.

---

## Part 5: Flash Memory & Safe Calibration

### 13. `Flash: own pages, magic + CRC validated, reloaded at boot`
* **What it means:**  
  Sensor calibration factors and cryptographic keys are written into **Internal Flash Bank 2 (Page 126)**.
* **Why Magic + CRC is needed:**  
  Flash can contain unprogrammed garbage (`0xFFFFFFFF`). The firmware prefixes data with a 32-bit "magic word" (e.g. `0xCAFE1234`) and a CRC-16 checksum. At boot, if the magic word and CRC match, it loads the saved calibration. If not, it falls back to safe factory defaults.

---

### 14. `Safety: system auto-halts (g_systemHalt) while coefficients are written`
* **The Fatal Risk:**  
  On ARM microcontrollers, erasing and writing internal flash memory temporarily pauses CPU bus memory access. If an interrupt fires to stream telemetry while flash is being written, the CPU generates an unrecoverable memory collision.
* **The Safety Interlock:**  
  Before writing calibration coefficients, the desktop app sends `halt on`. The firmware stops the 100 Hz timer, writes flash safely, and then `halt off` restarts the streaming.

---

## Part 6: Cryptographic Security

### 15. `Security: opt-in AES-256-GCM (HW AES + SW fallback)`
* **What it means:**  
  Protects wireless Bluetooth telemetry from being eavesdropped or modified by hackers.
* **Why AES-GCM?**  
  It provides both **Confidentiality** (encrypts the 20 bytes of sensor payload) and **Integrity** (generates an 8-byte authentication tag). If even one bit is tampered with in transit, the tag check fails and the packet is rejected.
* **HW AES + SW Fallback:**  
  The STM32U585 hardware AES peripheral does the encryption in microseconds. If the hardware self-test ever fails, an onboard software AES implementation automatically takes over, preventing communication loss.

---

## Part 7: Communication Recovery & CLI Console

### 16. `Console: rich CLI: mode/demo/set/get/sensors/halt/crypto`
* **What it means:**  
  The firmware includes a built-in terminal command line over UART. An engineer can type:
  * `sensors` $\rightarrow$ Prints real-time voltages and variance.
  * `mode real` / `mode demo` $\rightarrow$ Switches between real sensors and synthetic walking.
  * `set arm 0.25` $\rightarrow$ Changes the moment arm distance instantly.
  * `crypto on` $\rightarrow$ Arms hardware encryption.

---

### 17. `Link recovery: SLAVE_RxService() re-arms idle-DMA on error`
### 18. `Console recovery: RX errors clear ORE/FE/NE and re-arm`
* **The Problem:**  
  In noisy environments, electrostatic discharge (ESD) or unplugging cables causes UART hardware errors:
  * `ORE` (Overrun Error): Data arrived faster than CPU processed it.
  * `FE` (Framing Error): Missing stop bit due to baud noise.
  * `NE` (Noise Error): Spikes detected on RX line.
* **The Fix:**  
  Standard HAL drivers permanently shut down the UART receiver on any error. This recovery code detects `ORE/FE/NE`, clears the hardware error flags, and re-arms DMA listening immediately.

---

## Part 8: Desktop Studio (Electron Architecture)

### 19. `Main process: window, serial IPC, permissions, native dialogs, file writes`
* **What it means:**  
  Electron runs like a web browser with a backend Node.js server. The **Main Process** is the privileged backend. It directly talks to Windows COM ports, manages window creation, and writes CSV/JSON files to the hard drive.

---

### 20. `Preload: secure contextBridge API (no Node in renderer)`
* **Why it matters (Cybersecurity):**  
  The graphical UI (Renderer) runs HTML/JavaScript. If the UI had direct access to Node.js system calls, an infected charting library could execute malicious code on the clinician's computer.
* **The Bridge:**  
  A `preload.js` script creates a secure bridge (`contextBridge.exposeInMainWorld`). The UI can only call clean, safe functions (like `window.electronAPI.sendCommand()`), with zero access to low-level operating system internals.

---

### 21. `Parser: hello scan -> frame extract -> CRC -> field decode`
* **How incoming data is parsed:**
  1. **Hello Scan:** Scans incoming serial byte buffer looking for the 2-byte sync header (`0xAA 0x55`).
  2. **Frame Extract:** Once found, pulls the next 23 bytes (total 25 bytes).
  3. **CRC Check:** Computes CRC-16/MODBUS across bytes 0–22 and verifies it matches bytes 23–24. If corrupted, it discards the frame.
  4. **Field Decode:** Unpacks little-endian integers into floating-point numbers ($/100$) and sends them to the live dashboard and charts.
