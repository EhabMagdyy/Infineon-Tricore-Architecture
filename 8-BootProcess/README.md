# AURIX TC3xx Boot Process & HSM Architecture
### Part 8 — Boot ROM, Boot Firmware, BMHD Validation, HSM & Multicore Release

> Before a single line of application code runs, AURIX must answer a harder question
> than "what do I execute?" — it must answer *"can I trust what I'm about to execute?"*
> This is the complete journey from reset assertion through cryptographically validated
> firmware, Hardware Security Module initialization, lockstep configuration, and
> multicore release — the foundation every AURIX application silently depends on.
>
> **Document scope:** Covers the complete startup sequence of the Infineon AURIX TC3xx
> family from reset assertion through multicore application launch, including Boot ROM
> internals, Boot Firmware, Boot Mode Header (BMHD) structure and validation, all three
> boot modes, Hardware Security Module (HSM) integration, lockstep safety configuration,
> application startup software, and multicore release. Targeted at embedded systems
> engineers working with AURIX TC3xx derivatives (TC36x, TC37x, TC38x, TC39x).

---

## Table of Contents

| # | Topic |
|---|---|
| 1 | [Architecture Overview](#1--architecture-overview) |
| 2 | [TriCore CPU Subsystem](#2--tricore-cpu-subsystem) |
| 3 | [Reset Sources and Types](#3--reset-sources-and-types) |
| 4 | [Boot ROM (BROM)](#4--boot-rom-brom) |
| 5 | [Boot Firmware](#5--boot-firmware) |
| 6 | [CPU0 — The Boot Master](#6--cpu0--the-boot-master) |
| 7 | [Boot Mode Selection](#7--boot-mode-selection) |
| 8 | [Boot Mode Header (BMHD)](#8--boot-mode-header-bmhd) |
| 9 | [BMHD Validation Process](#9--bmhd-validation-process) |
| 10 | [Internal Flash Boot](#10--internal-flash-boot) |
| 11 | [Alternate Boot Mode (ABM)](#11--alternate-boot-mode-abm) |
| 12 | [Bootstrap Loader (BSL)](#12--bootstrap-loader-bsl) |
| 13 | [Hardware Security Module (HSM)](#13--hardware-security-module-hsm) |
| 14 | [Lockstep Configuration](#14--lockstep-configuration) |
| 15 | [Safety Checks During Boot](#15--safety-checks-during-boot) |
| 16 | [Application Startup Software](#16--application-startup-software) |
| 17 | [Multicore Startup](#17--multicore-startup) |
| 18 | [Watchdog Management During Boot](#18--watchdog-management-during-boot) |
| 19 | [Non-Volatile Memory Layout](#19--non-volatile-memory-layout) |
| 20 | [Complete Boot Sequence Diagram](#20--complete-boot-sequence-diagram) |
| 21 | [Common Boot Failures and Debugging](#21--common-boot-failures-and-debugging) |
| 22 | [AURIX Development Studio and Boot Debugging](#22--aurix-development-studio-and-boot-debugging) |
| 23 | [Summary and Key Principles](#23--summary-and-key-principles) |
| 24 | [Glossary](#24--glossary) |
| 25 | [References](#25--references) |

---

---

## 1. Architecture Overview

The Infineon AURIX TC3xx family is a 32-bit multicore microcontroller platform built on the **TriCore 1.6.2P** architecture. It is designed for high-reliability, safety-critical automotive applications requiring **ISO 26262 ASIL-D** compliance. The platform combines:

- A **RISC load-store architecture** with DSP extensions
- **Multiple TriCore CPUs** (up to 6 cores on TC39x derivatives)
- **Integrated Hardware Security Module (HSM)** for EVITA-compliant secure operations
- A **Flexible Peripheral Interconnect (FPI)** and **Shared Resource Interconnect (SRI)**
- Hardware lockstep, LBIST, ECC-protected memories, and redundant watchdogs

### 1.1 TC3xx Product Family

| Derivative | Cores | Program Flash | Data Flash | Typical Application |
|------------|-------|--------------|------------|---------------------|
| TC36x      | 3     | up to 4 MB   | 128 KB     | Body / powertrain   |
| TC37x      | 4     | up to 8 MB   | 384 KB     | ADAS / chassis      |
| TC38x      | 4     | up to 16 MB  | 384 KB     | Gateway / domain    |
| TC39x      | 6     | up to 16 MB  | 512 KB     | Central compute     |

### 1.2 Why Boot is Complex on AURIX

Unlike a simple microcontroller that directly executes from reset vector in flash, AURIX implements a **multi-phase boot architecture** because:

1. **Functional safety** requires hardware to self-check before running application code.
2. **Security** requires HSM authentication before trusting application binaries.
3. **Multicore** requires a defined master (CPU0) to initialize shared infrastructure before other cores are released.
4. **Flash technology** requires a startup sequence to bring NVM to operational state before code fetch can begin.
5. **Automotive reliability** requires redundant configuration storage with CRC protection.

The result is that from power-on to `main()` there can be several milliseconds of controlled initialization — all of it deterministic, auditable, and certified.

---

## 2. TriCore CPU Subsystem

### 2.1 TriCore 1.6.2P Architecture

Each TriCore core unifies three processor architectures in one pipeline:

- **RISC controller core** — 32-bit load/store with 16-bit compressed instruction encoding
- **DSP signal-processing core** — saturating arithmetic, multiply-accumulate, fractional operations
- **Real-time OS core** — hardware context management, interrupt nesting, trap handling

All instructions execute in **1 clock cycle** in the common case. Branch instructions cost 1–3 cycles with dynamic branch prediction.

### 2.2 Memory Architecture

The TC3xx memory system is segmented into:

```
Address Map (simplified)
────────────────────────────────────────────────────────
0x00000000 - 0x0FFFFFFF   Program Flash (PFlash) banks
0x10000000 - 0x1FFFFFFF   Data Flash (DFlash/UCB)
0x40000000 - 0x4FFFFFFF   SRI Peripherals
0x50000000 - 0x5FFFFFFF   FPI Peripherals
0x70000000 - 0x70FFFFFF   Boot ROM (BROM) — CPU0 reset vector
0x80000000 - 0x8FFFFFFF   CPU0 DSPR (Data Scratchpad RAM)
0x90000000 - 0x9FFFFFFF   CPU1 DSPR
0xA0000000 - 0xAFFFFFFF   CPU2 DSPR
0xC0000000 - 0xCFFFFFFF   CPU0 PSPR (Program Scratchpad RAM)
0xD0000000 - 0xDFFFFFFF   CPU1 PSPR
0xE0000000 - 0xEFFFFFFF   CPU2 PSPR
0xF0000000 - 0xFFFFFFFF   LMU / Global RAM / Registers
```

Each core has its own private scratchpad RAM (PSPR for instructions, DSPR for data). The CPU0 PSPR is important during Bootstrap Loader operation — downloaded code lands here.

### 2.3 CSA — Context Save Area

The TC3xx does **not** use a conventional hardware stack for interrupt/trap context. Instead it uses a linked list of **Context Save Areas (CSAs)** in memory, managed by hardware. Each CSA entry saves the full upper and lower context of a CPU task.

During boot startup software, the CSA free list must be initialized before any function call, interrupt, or trap can safely occur. This is one of the mandatory steps in startup code before `main()`.

### 2.4 Pipeline and Instruction Set

- 32-bit and 16-bit (compact) instruction formats
- 4 Gbyte flat address space (32-bit)
- Hardware floating-point unit (FPU) including double-precision on TC3xx
- Virtualization support (guest/host mode)
- Automatic context save on interrupt entry, restore on exit

---

## 3. Reset Sources and Types

### 3.1 Reset Hierarchy

AURIX implements a layered reset architecture. Different reset types affect different scopes of the device:

```
PORST (Power-On Reset)
  └── System Reset
        └── Application Reset
              └── CPU Reset (per-core)
                    └── Debug Reset
```

Each layer resets everything in its own scope plus all layers below it.

### 3.2 Power-On Reset (PORST)

**Trigger:** PORST pin driven low, or supply voltage drops below threshold (under-voltage detection).

**Scope:** The widest possible reset. Resets the entire device including:
- All CPU cores (halt all, restart CPU0 at BROM entry point)
- All peripheral registers
- PLL / clock system
- EVR (Embedded Voltage Regulator)
- Flash state machine
- SMU (Safety Management Unit)

**Memory state after PORST:** SRAM contents are **undefined** — not initialized. Code must never assume SRAM holds valid data after a cold power-on reset.

**Boot firmware entry:** Always entered. Full initialization sequence runs.

### 3.3 System Reset (KRST)

**Trigger:** Software writes to `SCU_RSCON` register, or a safety violation forces a system reset via the SMU.

**Scope:** Resets CPU cores, most peripherals, and the clock system. Does **not** reset the EVR.

**Memory state:** SRAM contents may be retained depending on specific reset cause. Certain registers in the SCU survive to record the reset cause.

**Boot firmware entry:** Entered. Firmware evaluates the reset type register to determine what initialization is needed.

### 3.4 Application Reset

**Trigger:** Software request via `SCU_SWRSTCON.SWRSTREQ`, or triggered by specific SMU alarm actions.

**Scope:** Similar to system reset but leaves debug infrastructure active. Common in production software for watchdog recovery or firmware update completion.

**Boot firmware entry:** Entered. This is the reset type most commonly triggered during OTA update flows after writing new application code.

### 3.5 CPU Reset (Kernel Reset)

**Trigger:** Write to `CPUx_KRST0` / `CPUx_KRST1` registers (requires both written in sequence for security).

**Scope:** Resets a single CPU core and its local resources (pipeline, local SRAM, local timers). Other cores are unaffected.

**Boot firmware entry:** CPU0 kernel reset does re-enter boot firmware. Other CPU resets restart at the address stored in their start core register.

**Use case:** Fault recovery in safety-critical systems. If CPU1 suffers a trap or watchdog timeout, CPU0 can reset only CPU1 and restart its task without disturbing the rest of the system.

### 3.6 Reset Cause Register

After any reset, firmware reads `SCU_RSTSTAT` to determine the source:

| Bit Field     | Meaning                          |
|---------------|----------------------------------|
| `PORST`       | Power-on reset                   |
| `CB0`         | OCDS (debug) system reset        |
| `CB1`         | OCDS CPU0 reset                  |
| `CB3`         | OCDS application reset           |
| `TP`          | Trap reset                       |
| `EVR13`       | 1.3 V regulator under-voltage    |
| `EVR33`       | 3.3 V regulator under-voltage    |
| `STBYR`       | Standby regulator watchdog       |
| `LBIST`       | LBIST-triggered reset            |

Boot firmware uses this information to skip unnecessary re-initialization (e.g., no need to re-run LBIST on a software application reset) and to log fault events.

---

## 4. Boot ROM (BROM)

### 4.1 What the BROM Is

The Boot ROM is a **read-only memory programmed by Infineon during manufacturing**. It is physically located in internal ROM, mapped at the CPU0 reset vector address. It is the first code CPU0 executes on any reset — there is no earlier user-accessible code path.

The BROM cannot be modified, reprogrammed, or bypassed by user software. Its content is fixed for the lifetime of the device.

### 4.2 BROM Location and Reset Vector

On TC3xx devices:

- CPU0 reset vector: `0x80000020` (CPU0 PSPR area, but initially fetched from BROM internal ROM)
- BROM internal address: `0x00000000` (internal ROM, mirrored/cached as needed)

The exact physical mapping is device-specific and documented in the TC3xx derivative-specific User Manual appendix.

### 4.3 BROM Responsibilities

The Boot ROM performs the following steps in strict order:

1. **Minimal CPU initialization** — sets pipeline to a known state
2. **EVR stabilization check** — waits for embedded voltage regulators (1.3 V core, 3.3 V I/O) to settle within specification
3. **Flash startup sequence** — brings the Program Flash and Data Flash state machines to operational state so that read/write/erase commands can be issued within the full specified operating range
4. **UCB read** — reads the User Configuration Block from Data Flash to determine boot configuration
5. **Hand-off to Boot Firmware** — jumps into the Boot Firmware code, also stored in BROM

### 4.4 PSFDM — Power Supply Friendly Debug Monitor

The BROM contains a routine called the **Power Supply Friendly Debug Monitor (PSFDM)**. Its purpose is twofold:

**Problem 1 — Current spike on CPU release:** When an OCDS debugger halts multiple CPU cores simultaneously and then releases them, all cores begin fetching instructions at the same moment. On AURIX TC3xx with 4–6 cores, this causes a near-instantaneous current surge that can exceed the EVR's load regulation capability, causing a momentary supply droop.

**Problem 2 — EVR overshoot on CPU halt:** When multiple active CPUs are halted by the debugger simultaneously, the sudden drop in current demand can cause the EVR output to momentarily overshoot.

PSFDM mitigates both by staggering the halt/release transitions and monitoring the EVR output voltage. This is transparent to application developers but critical for stable debug sessions, especially on hardware bringup where EVR tuning may not yet be finalized.

### 4.5 BROM Limitations

- No user code executes before BROM completes
- BROM cannot be observed or instrumented (no OCDS breakpoints within BROM)
- BROM execution time is not user-configurable
- On some TC3xx variants, LBIST (Logic Built-In Self Test) is triggered from BROM before Boot Firmware runs

---

## 5. Boot Firmware

### 5.1 Definition and Location

Boot Firmware is software **stored within the Boot ROM** (same physical ROM as BROM, but a distinct logical section). It is Infineon-programmed and immutable. Boot Firmware is the first code that performs meaningful device configuration decisions based on user-programmable configuration data (the BMHD).

The distinction between BROM and Boot Firmware is conceptual:
- **BROM** = lowest-level hardware bring-up (EVR, flash startup)
- **Boot Firmware** = configuration evaluation and mode selection

### 5.2 When Boot Firmware Executes

Boot Firmware is entered after any of the following reset types:

- PORST pin reset assertion (power-on)
- System reset (KRST)
- Application reset
- CPU0 kernel reset

In all cases, **CPU0 executes Boot Firmware**. Other CPUs remain in HALT.

### 5.3 Boot Firmware Execution Flow

```
Boot Firmware Entry
        │
        ▼
Read SCU_RSTSTAT
(determine reset cause)
        │
        ▼
Read UCB_BMHD0..3
(load all BMHD pairs)
        │
        ▼
Validate BMHD pairs
(CRC, inverted CRC, ID field)
        │
        ├─── Valid BMHD found ──────────────────────────────┐
        │                                                    │
        ▼                                                    ▼
Configure from BMHD:                           No valid BMHD:
  - Lockstep enable/disable                   Enter BSL or error state
  - HSM enable/disable
  - Debug port lock
  - Startup address (STAD)
        │
        ▼
Start HSM (if enabled)
        │
        ▼
Configure Lockstep
(CPU0 + shadow core)
        │
        ▼
Evaluate Boot Mode:
  Internal Flash / ABM / BSL
        │
        ├── Internal Flash ── Jump to STAD in PFlash
        │
        ├── ABM ─────────── Verify ABM header + CRC → Jump to alternate image
        │
        └── BSL ─────────── Initialize ASC/CAN interface → Receive code → Jump to CPU0_PSPR
```

### 5.4 Boot Firmware vs. User Bootloader

It is important to distinguish between Infineon's Boot Firmware (in BROM) and a user-written bootloader (in PFlash):

| Property                    | Boot Firmware (BROM)       | User Bootloader (PFlash)         |
|-----------------------------|----------------------------|----------------------------------|
| Location                    | Internal ROM               | Program Flash                    |
| Modifiable                  | No                         | Yes                              |
| Executes before             | Everything                 | Application only                 |
| Handles BMHD                | Yes                        | No (already past BMHD)           |
| Can update PFlash           | Via BSL mode only          | Yes, full freedom                |
| Can run on all cores        | CPU0 only                  | Any core                         |
| ASIL certification          | Infineon-certified         | User responsibility               |

A user-written OTA bootloader lives in PFlash and is launched by Boot Firmware after BMHD validation points `STAD` to the bootloader's entry point.

### 5.5 Startup Sequence Summary from AP32381

Per the official Infineon application note AP32381, the start-up sequence for TC3xx is divided into:

1. PSW register initialization (Interrupt Stack Pointer, User-1 mode, maximum Call Depth Counter)
2. Reset evaluation (determine reset cause from `SCU_RSTSTAT`)
3. Flash start and UCB read
4. BMHD evaluation and validation
5. Boot mode determination
6. HSM start (if configured)
7. Lockstep configuration (if configured)
8. Jump to application start address (STAD)

---

## 6. CPU0 — The Boot Master

### 6.1 Exclusive Boot Execution

After any reset that affects the full device, **only CPU0 begins executing**. All other TriCore cores (CPU1, CPU2, ... CPUn) are held in `HALT` state by hardware. They cannot fetch instructions, access memory, or interact with peripherals.

This is enforced by the **SCU (System Control Unit)** and the per-core `CORE_SEL` / start core registers. CPU0 is hardwired as the boot master — this cannot be changed by software.

### 6.2 Why CPU0 Exclusively?

Several safety-critical reasons drive the single-master boot design:

1. **Initialization order** — shared resources (PLL, memory, SMU, HSM) must be configured exactly once before any other code touches them. Two cores simultaneously initializing the PLL would cause unpredictable behavior.

2. **Safety infrastructure first** — lockstep must be configured before application code runs on any core. If CPU1 started in parallel with CPU0, it might execute application code before lockstep is active, violating ASIL-D requirements.

3. **Determinism** — a fixed single-master startup sequence is auditable and certifiable. Parallel initialization with shared resource contention is non-deterministic and therefore not certifiable.

4. **Debug simplicity** — during hardware bringup, knowing that exactly one core executes simplifies OCDS debugging enormously.

### 6.3 CPU0 During Boot — State Machine

```
RESET ASSERTED
      │
      ▼ (CPU0 only starts)
┌─────────────────────┐
│   Boot ROM / BROM   │
└─────────────────────┘
      │
      ▼
┌─────────────────────────────┐
│   Boot Firmware             │
│  - Read BMHD                │
│  - Validate config          │
│  - Configure HSM            │
│  - Configure lockstep       │
│  - Select boot mode         │
└─────────────────────────────┘
      │
      ▼
┌─────────────────────────────┐
│   Application Startup Code  │   (at STAD address in PFlash)
│  - Init stack, CSA          │
│  - Init memory              │
│  - Init clock / PLL         │
│  - Init peripherals         │
│  - Init RTOS / OS           │
└─────────────────────────────┘
      │
      ▼
┌─────────────────────────────┐
│   Release CPU1..N           │   (write start addresses to CORECNTRL registers)
└─────────────────────────────┘
      │
      ▼
┌─────────────────────────────┐
│   CPU0 Application Task     │
└─────────────────────────────┘
```

### 6.4 HALT State Details

Cores in HALT state:
- Do not fetch or execute instructions
- Do not respond to interrupts
- Do not generate bus traffic
- Consume minimal power (clock-gated)
- Cannot self-release — only CPU0 (or an authorized master) can release them

A core exits HALT when CPU0 writes its **start address** to the corresponding core control register and sets the `BHALT` bit appropriately.

---

## 7. Boot Mode Selection

### 7.1 Overview

After BMHD validation, Boot Firmware must decide **where to get the executable code from**. This is the boot mode decision. There are three primary modes:

| Mode                   | Code Source               | Typical Use               |
|------------------------|---------------------------|---------------------------|
| Internal Flash Boot    | Program Flash (PFlash)    | Normal production run     |
| Alternate Boot Mode    | Alternate PFlash region   | OTA update / recovery     |
| Bootstrap Loader (BSL) | External host via UART/CAN | Factory programming       |

### 7.2 Selection Priority

Boot mode is determined by evaluating in priority order:

1. **BMHD BOOTMODE field** — primary control, set during flash programming
2. **Hardware configuration pins** — certain TC3xx pins sampled at reset can force BSL mode regardless of BMHD (used for factory bring-up before BMHD is programmed)
3. **BMHD validation failure** — if all BMHD pairs are invalid, Boot Firmware falls back to BSL mode to allow recovery

### 7.3 BOOTMODE Field Values

The `BOOTMODE` field in BMHD encodes the desired boot path:

| Value | Mode                       |
|-------|----------------------------|
| `00`  | Internal start from flash (STAD) |
| `01`  | Alternate Boot Mode (ABM)  |
| `10`  | Bootstrap Loader via ASC (UART) |
| `11`  | Bootstrap Loader via CAN   |

---

## 8. Boot Mode Header (BMHD)

### 8.1 Purpose and Location

The **Boot Mode Header (BMHD)** is the primary user-programmable configuration structure for the TC3xx boot process. It is stored in the **UCB (User Configuration Block)** area of **Data Flash**.

The UCB is a special protected region of DFlash that stores device configuration data including:
- Boot Mode Headers (BMHD0–BMHD3)
- Protection configuration (read/write/erase locks)
- Tuning parameters
- HSM configuration

The UCB is written during device programming (factory or OTA update) and is protected from accidental modification.

### 8.2 BMHD Redundancy Structure

TC3xx defines **four independent BMHD sets**, numbered BMHD0 through BMHD3. Each set contains two copies:

```
Data Flash — UCB Area
├── UCB_BMHD0_ORIG   (original copy of BMHD set 0)
├── UCB_BMHD0_COPY   (redundant copy of BMHD set 0)
├── UCB_BMHD1_ORIG
├── UCB_BMHD1_COPY
├── UCB_BMHD2_ORIG
├── UCB_BMHD2_COPY
├── UCB_BMHD3_ORIG
└── UCB_BMHD3_COPY
```

Boot Firmware evaluates BMHD0 first, then BMHD1, BMHD2, BMHD3 in order. Within each set, ORIG is tried first, then COPY. The first valid pair found is used.

**At minimum, one valid BMHD pair (ORIG + COPY) must be programmed** for normal internal flash boot to work.

### 8.3 BMHD Data Structure

Each BMHD instance is a fixed-size structure containing the following fields:

| Field         | Size    | Description                                                    |
|---------------|---------|----------------------------------------------------------------|
| `BMHDID`      | 16 bits | Magic identifier — must equal `0xB359` for valid BMHD         |
| `STAD`        | 32 bits | Start address of user application in PFlash                    |
| `CRCBMHD`     | 32 bits | CRC-32 checksum of the BMHD structure                          |
| `CRCBMHDINV`  | 32 bits | Bitwise inversion of `CRCBMHD` (integrity double-check)        |
| `BOOTMODE`    | 2 bits  | Boot mode selection (internal / ABM / BSL-ASC / BSL-CAN)      |
| `HWCFG`       | bits    | Hardware configuration (lockstep, debug port, HSM enable)      |
| `ABM`         | 1 bit   | Alternate Boot Mode enable flag                                |
| `DBG`         | 1 bit   | Debug port enable (OCDS)                                       |
| `LOCKSTEP`    | 1 bit   | CPU0 lockstep enable                                           |
| `HSM`         | 1 bit   | HSM startup enable                                             |
| `CHKSTART`    | 32 bits | Start address of application CRC range (ABM verification)      |
| `CHKEND`      | 32 bits | End address of application CRC range                           |
| `CHKRESULT`   | 32 bits | Expected CRC result of application memory range                |

### 8.4 STAD — Start Address Field

The `STAD` field is the most critical field in the BMHD. It contains the **physical address of the first instruction of user code** in Program Flash. After all validation passes, Boot Firmware performs an unconditional jump to this address.

Requirements for `STAD`:
- Must be within a valid PFlash address range
- Must be aligned to the instruction fetch granularity (typically 4-byte aligned)
- Must point to valid, executable code (if it points to garbage, a CPU trap occurs immediately)

Common `STAD` values:
- `0x80000020` — PFlash bank 0, used in simple single-image setups
- Custom addresses — used in A/B partition OTA schemes where bootloader lives at a fixed offset

### 8.5 BMHD Programming

BMHD structures are written to the UCB during:

1. **Factory device programming** — production flash programmer writes BMHD as part of the initial image
2. **OTA firmware update** — the update software must re-program UCB_BMHD after writing the new application to update `STAD` and `CHKRESULT`
3. **Development** — AURIX Development Studio or a programmer tool writes BMHD as part of debug session setup

**Important:** Writing to UCB requires unlocking the UCB protection and follows the Data Flash program/erase protocol. A partial write that corrupts `CRCBMHD` will cause that BMHD to fail validation and Boot Firmware will fall through to the next pair.

---

## 9. BMHD Validation Process

### 9.1 Full Validation Sequence

Boot Firmware validates each BMHD instance through a strict 4-stage process. All four stages must pass. If any stage fails, that BMHD instance is discarded and the next one is tried.

```
For each BMHD instance (BMHD0_ORIG, BMHD0_COPY, BMHD1_ORIG, ...):

Stage 1: BMHDID check
  ├── Read BMHDID field from UCB flash
  ├── Compare against expected magic value 0xB359
  ├── PASS → proceed to Stage 2
  └── FAIL → discard this BMHD, try next

Stage 2: CRC computation
  ├── Compute CRC-32 over the BMHD structure bytes
  ├── Compare computed CRC against stored CRCBMHD field
  ├── PASS → proceed to Stage 3
  └── FAIL → discard this BMHD, try next

Stage 3: Inverted CRC check
  ├── Read CRCBMHDINV field
  ├── Verify that CRCBMHDINV == bitwise_NOT(CRCBMHD)
  ├── PASS → proceed to Stage 4
  └── FAIL → discard this BMHD, try next

Stage 4: STAD range check
  ├── Verify STAD is within a valid PFlash address range
  ├── Verify STAD alignment
  ├── PASS → use this BMHD configuration
  └── FAIL → discard this BMHD, try next
```

### 9.2 Why the Inverted CRC?

The dual CRC + inverted-CRC scheme provides protection against two distinct failure modes:

1. **Stuck-at-0 fault** — if a memory cell is stuck at logic 0, all stored data reads as 0x00000000. A CRC of all-zeros happens to be a valid CRC for specific data patterns. The inverted copy (which would be 0xFFFFFFFF for an all-zeros original) catches this case.

2. **Stuck-at-1 fault** — symmetric protection: if a cell is stuck at 1, `CRCBMHD` would be 0xFFFFFFFF, but `CRCBMHDINV` would be 0x00000000, not 0xFFFFFFFF. Mismatch detected.

3. **Single event upset (SEU)** — cosmic ray or alpha particle flips a bit. Either the CRC no longer matches its computed value, or the inverted copy no longer matches its expected relationship. Either way, fault detected.

This dual-word protection pattern is a standard technique in automotive NVM safety design (referenced in IEC 60730 and ISO 26262).

### 9.3 BMHD Fallback Behavior

| Scenario | Boot Firmware Action |
|----------|---------------------|
| BMHD0_ORIG valid | Use BMHD0_ORIG, boot from STAD |
| BMHD0_ORIG invalid, BMHD0_COPY valid | Use BMHD0_COPY, boot from STAD |
| Both BMHD0 instances invalid | Try BMHD1 |
| All 8 instances (BMHD0–3 × 2) invalid | Fall back to BSL mode for recovery |

The fallback to BSL when all BMHD pairs fail is a deliberate safety net that allows factory recovery of a device with corrupted configuration. Without it, a device with a bad UCB write would be permanently bricked.

---

## 10. Internal Flash Boot

### 10.1 Overview

Internal Flash Boot is the standard production boot path. After a valid BMHD is found and the `BOOTMODE` field selects internal flash, Boot Firmware jumps directly to the `STAD` address in Program Flash (PFlash).

```
Boot Firmware
     │
     ▼
Read STAD from BMHD
     │
     ▼
Jump to STAD address
     │
     ▼
User application startup code
(at STAD in PFlash Bank 0)
```

### 10.2 Program Flash (PFlash) Organization

TC3xx PFlash is organized into multiple banks, each subdivided into sectors:

- **Physical sectors** — the erase granularity (typically 16 KB or 64 KB depending on derivative)
- **Logical sectors** — software view used by the flash driver
- **Banks** — independently erasable/programmable groups (Bank 0, Bank 1, ...)

For OTA A/B partitioning, the application is typically split across two regions:
- **Slot A** — active partition (STAD points here normally)
- **Slot B** — inactive partition (written during update, BMHD updated to point here after verification)

### 10.3 Flash ECC During Boot

All PFlash accesses during code fetch go through the flash ECC hardware. TC3xx uses a Hamming-based ECC scheme that:
- **Detects** 2-bit errors (Double Bit Error — DED)
- **Corrects** 1-bit errors (Single Bit Error — SEC) transparently

If an uncorrectable flash ECC error occurs during instruction fetch at boot time, the flash controller generates a **bus error** which causes CPU0 to take a bus error trap. This is the expected behavior — it prevents execution of corrupted code.

---

## 11. Alternate Boot Mode (ABM)

### 11.1 Purpose

The Alternate Boot Mode provides a second, independently-verified application image path. It is designed for:

- **OTA firmware update staging** — write new firmware to the alternate region, verify it, then switch to it
- **Fallback / recovery image** — a known-good backup that can be activated if the primary image fails
- **Manufacturing test image** — a factory test program that can be run before releasing the device to normal operation

### 11.2 ABM Header Structure

The ABM region in PFlash starts with an **ABM header** — a structure placed by the programmer at the beginning of the alternate image region. The ABM header contains:

| Field        | Description                                              |
|--------------|----------------------------------------------------------|
| `ABMID`      | Magic identifier verifying this is a valid ABM header    |
| `ABMSTAD`    | Entry point of the alternate application                 |
| `CHKSTART`   | Start address of the memory range to CRC-check           |
| `CHKEND`     | End address of the memory range to CRC-check             |
| `CHKRESULT`  | Expected CRC-32 result of the memory range               |
| `CRCABMHD`   | CRC of the ABM header itself                             |
| `CRCABMHDINV`| Inverted CRC of the ABM header (same dual-CRC protection)|

### 11.3 ABM Verification Flow

```
Boot Firmware detects ABM mode
          │
          ▼
Read ABM header from PFlash
(at defined ABM region start address)
          │
          ▼
Validate ABM header ID field
          │
          ▼
Verify ABM header CRC + inverted CRC
          │
          ▼
Compute CRC over application memory
(from CHKSTART to CHKEND)
          │
          ▼
Compare computed CRC with CHKRESULT
          │
  ┌───────┴────────┐
  ▼                ▼
PASS             FAIL
  │                │
Jump to         Return to normal boot
ABMSTAD         (or enter BSL recovery)
```

### 11.4 ABM and OTA Update Workflow

A typical A/B OTA update using ABM works as follows:

1. **Device is running** application in PFlash Bank 0 (BMHD0 STAD = `0x80000000`).
2. **OTA agent** receives new firmware, writes it to PFlash Bank 1 (the alternate region).
3. **OTA agent** computes CRC of the newly written Bank 1 image.
4. **OTA agent** writes a valid ABM header at the start of Bank 1 with correct `CHKRESULT`.
5. **OTA agent** reprograms UCB BMHD to set `BOOTMODE = ABM`.
6. **OTA agent** triggers an application reset.
7. **Boot Firmware** reads BMHD, sees ABM mode.
8. **Boot Firmware** verifies ABM header and CRC of Bank 1 image.
9. On **success**: jumps to Bank 1 application.
10. On **failure**: reverts BMHD to normal flash boot, boots original Bank 0 image.

---

## 12. Bootstrap Loader (BSL)

### 12.1 Purpose

The Bootstrap Loader mode is the lowest-level programming interface of AURIX. It allows an external host (a PC, an ECU programmer, or a production test fixture) to download code into the device when no valid application is present in PFlash.

BSL is typically used for:
- **Initial factory programming** of blank devices
- **Recovery** of devices whose PFlash has been corrupted
- **Low-level debug** before the application firmware is ready

### 12.2 Supported Interfaces

TC3xx BSL supports two communication interfaces:

| Interface | Protocol | Physical Layer | Typical Use |
|-----------|----------|----------------|-------------|
| ASC BSL   | UART (asynchronous serial) | UART0 peripheral | PC-based programmer tools |
| CAN BSL   | CAN 2.0B | CAN0 peripheral | In-vehicle ECU programming |

The active interface is selected by the `BOOTMODE` field in BMHD (or by hardware pin state if BMHD is not programmed).

### 12.3 BSL Operation

```
Boot Firmware selects BSL mode
           │
           ▼
Initialize selected interface
(ASC UART or CAN at defined baud rate)
           │
           ▼
Wait for host connection
(timeout → may revert to internal boot)
           │
           ▼
Receive code packets from host
(with checksums / ACK protocol)
           │
           ▼
Write received code into CPU0_PSPR
(Program Scratchpad RAM of CPU0)
           │
           ▼
Host sends EXECUTE command
           │
           ▼
Jump to downloaded code in CPU0_PSPR
           │
           ▼
Downloaded code runs
(typically: flash programmer, memory test,
or stub bootloader that programs PFlash)
```

### 12.4 CPU0_PSPR as BSL Target

Downloaded code lands in **CPU0_PSPR** (CPU0 Program Scratchpad RAM), not directly in PFlash. This is intentional:

1. PSPR is RAM — it does not require erase-before-write, so small code payloads can be written quickly
2. Code in PSPR executes at full CPU speed without flash access latency
3. The downloaded stub code typically then calls flash write routines to program PFlash with the full firmware image
4. PSPR is cleared on any cold reset, preventing residual BSL payloads from surviving power cycles

### 12.5 BSL Security Considerations

BSL mode represents a significant security attack surface. An adversary with physical UART/CAN access during boot could:
- Download arbitrary code to PSPR
- Execute that code with full CPU privileges
- Read out flash contents, disable protection, or overwrite firmware

TC3xx mitigates this through:
1. **Hardened BMHD** — BSL mode is only entered if explicitly configured in BMHD or if BMHD is invalid. Production devices with a valid BMHD pointing to internal flash never enter BSL mode unless forced.
2. **Protection bits in UCB** — read/write protection on PFlash and DFlash sectors can be configured to prevent unauthorized access even from BSL-downloaded code.
3. **HSM authentication** — on HSM-enabled devices, BSL mode can be configured to require authenticated commands before allowing flash operations.
4. **Debug port lock** — the OCDS debug interface can be permanently disabled via UCB settings, preventing debugger-assisted flash readout.

---

## 13. Hardware Security Module (HSM)

### 13.1 Overview

The Hardware Security Module is an independent, isolated security processor integrated into the AURIX TC3xx die. It implements the **EVITA (E-safety Vehicle Intrusion Protected Applications)** full HSM specification — the most comprehensive of three EVITA hardware security levels (Light, Medium, Full).

The HSM is:
- **Physically isolated** — separate CPU, RAM, ROM, and peripherals inaccessible to TriCore CPUs
- **Cryptographically capable** — dedicated hardware engines for AES, PKI (RSA, ECC), TRNG, hash functions
- **Key-isolated** — secret keys never leave the HSM boundary in plaintext
- **Independently operational** — the HSM can run its own firmware independent of the main TriCore cores

### 13.2 HSM Architecture

```
AURIX TC3xx — HSM Subsystem
┌─────────────────────────────────────────────────────┐
│  HSM CPU (32-bit RISC core, separate from TriCore)  │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │
│  │  HSM ROM    │  │  HSM RAM    │  │  HSM Flash │  │
│  │(boot code)  │  │ (working)   │  │ (firmware) │  │
│  └─────────────┘  └─────────────┘  └────────────┘  │
│                                                     │
│  Cryptographic Engines:                             │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐ │
│  │ AES  │ │ RSA  │ │ ECC  │ │ SHA  │ │  TRNG    │ │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────────┘ │
│                                                     │
│  Key Store (hardware-protected, never DMA-able)     │
│                                                     │
│  HSM ↔ TriCore Communication Mailbox               │
└─────────────────────────────────────────────────────┘
```

### 13.3 HSM Boot Sequence

The HSM follows its own internal boot sequence, triggered by the main Boot Firmware:

```
TC3xx Boot Firmware
        │
        ▼ (if BMHD.HSM = 1)
Start HSM subsystem
        │
        ▼
HSM ROM executes
(Infineon-programmed, immutable)
        │
        ▼
HSM ROM authenticates HSM firmware
(in HSM Flash area of Data Flash)
        │
        ▼
    ┌───┴────┐
    │Valid?  │
    └───┬────┘
   Yes  │   No
    ▼   │   ▼
HSM fw  │  HSM enters error state
runs    │  (TriCore boot may continue
    ▼   │  or halt, depending on config)
HSM ready signal
        │
        ▼
Boot Firmware continues
TriCore startup sequence
```

The HSM has its own internal validation similar to BMHD — HSM firmware in HSM Flash is authenticated via a hardware signature verification before the HSM CPU runs it. This forms the hardware root of trust.

### 13.4 Secure Boot with HSM

Secure boot is the process of verifying that the TriCore application code is authentic (signed by an authorized party) before it is executed. On TC3xx with HSM:

#### Step-by-step Secure Boot Flow:

1. **Key provisioning (factory):** A device-specific or OEM-specific public key is stored in the HSM Key Store during manufacturing. The corresponding private key never enters the device.

2. **Firmware signing (build time):** The application binary is signed by the OEM using the private key. The signature (typically ECDSA or RSA-PKCS1.5 over SHA-256 of the firmware) is appended to the firmware image.

3. **Boot time — HSM ROM runs:** HSM ROM starts, reads HSM firmware from HSM Flash, verifies its own firmware signature using the built-in Infineon root key.

4. **Boot time — HSM firmware runs:** HSM firmware reads the TriCore application image from PFlash (via the HSM's DMA access to the main bus). It computes SHA-256 of the application and verifies the OEM signature using the stored public key.

5. **Decision:**
   - **Signature valid:** HSM signals "boot authorized" to Boot Firmware. Boot Firmware jumps to STAD.
   - **Signature invalid:** HSM signals "boot denied". Boot Firmware can be configured to halt, enter BSL recovery, or trigger a system reset.

6. **Post-boot:** HSM continues running its security services stack (secure communication, key management, runtime integrity monitoring) while TriCore cores run the application.

### 13.5 HSM Security Services

Once the HSM is initialized and running its firmware, it provides services to the TriCore application via a **mailbox interface** (shared memory region with hardware access control):

#### 13.5.1 Symmetric Cryptography

- **AES-128/192/256** — ECB, CBC, CTR, GCM, CMAC modes
- Used for: data encryption at rest, secure communication payload encryption, key derivation

#### 13.5.2 Asymmetric Cryptography

- **RSA** — up to 4096-bit, PKCS#1 v1.5, OAEP, PSS padding
- **ECC** — NIST curves P-256, P-384, Brainpool; ECDSA signing/verification, ECDH key agreement
- Used for: firmware signature verification, certificate chain validation, session key establishment

#### 13.5.3 Hash Functions

- **SHA-1, SHA-2 (SHA-224, SHA-256, SHA-384, SHA-512)** — hardware-accelerated
- **HMAC** — keyed-hash message authentication
- Used for: integrity checks, PRF for key derivation, certificate digests

#### 13.5.4 True Random Number Generator (TRNG)

- Physical entropy source based on semiconductor noise
- Output conditioned through AES-CTR_DRBG (NIST SP 800-90A compliant)
- Used for: nonce generation, session keys, key material generation

#### 13.5.5 Key Management

- **Key Store** — isolated hardware memory inaccessible from TriCore side
- Keys can be marked as **non-exportable** (never leave HSM boundary)
- Keys can be **usage-restricted** (sign-only, verify-only, encrypt-only)
- **Key Update Protocol** — secure mechanism to update keys over a protected channel

#### 13.5.6 Application-Level Security Services

| Service | Description |
|---------|-------------|
| **Secure Boot** | Authenticate application before first execution |
| **Secure Flash Update** | Authenticate OTA firmware package before applying |
| **Secure Communication** | TLS/DTLS handshake acceleration, message authentication |
| **Immobilizer** | Vehicle immobilizer logic with key authentication |
| **Mileage Protection** | Tamper-evident odometer via HSM-signed monotonic counter |
| **Component Protection** | Authenticate ECU identity to vehicle network |
| **Tuning Protection** | Detect unauthorized firmware modification (chiptune) |
| **IP Protection** | Prevent reverse engineering of proprietary algorithms |

### 13.6 HSM Mailbox Interface

The TriCore application communicates with the HSM via a **mailbox** in shared SRAM. The protocol is:

```
TriCore Application                          HSM
      │                                       │
      │── Write command + parameters ──────►  │
      │   to mailbox input buffer             │
      │                                       │
      │── Set "command pending" flag ───────► │
      │                                       │
      │                               Process command
      │                               Execute crypto engine
      │                               Write result to output buffer
      │                                       │
      │◄─ "result ready" interrupt ──────────  │
      │                                       │
      │── Read result from mailbox ────────►  │
      │   output buffer                       │
```

The mailbox is access-controlled: the input buffer is **write-only** for TriCore, and the output buffer is **read-only** for TriCore. HSM has full access to both sides. This prevents TriCore code from directly inspecting intermediate crypto state.

### 13.7 HSM Firmware Update

The HSM runs its own firmware, which can be updated separately from the TriCore application. HSM firmware update requires:

1. New HSM firmware signed by Infineon (HSM firmware is Infineon IP, not user-programmable in the traditional sense)
2. Secure channel to deliver the signed update package
3. HSM's own update protocol to verify and install the new firmware

On TC3xx, HSM firmware is stored in a dedicated region of Data Flash protected by the HSM itself — TriCore CPUs cannot write to this region directly.

### 13.8 EVITA Compliance

EVITA Full HSM compliance (which TC3xx's HSM achieves) requires:

- Hardware isolation of security-critical functions
- Tamper detection (active and passive)
- Side-channel attack resistance for cryptographic engines
- Secure key storage with anti-cloning measures
- Protected TRNG with continuous health tests
- Secure boot for the HSM's own firmware
- Monotonic counters for replay protection

These requirements are documented in the EVITA D3.3 specification and are relevant for compliance with UN Regulation No. 155 (cyber security) and ISO/SAE 21434.

---

## 14. Lockstep Configuration

### 14.1 What is Lockstep?

Lockstep is a **hardware redundancy mechanism** for CPU execution safety. In lockstep mode, a **shadow core** executes the same instruction stream as the primary CPU, one cycle offset behind. After each instruction, the outputs of both cores are compared. Any discrepancy triggers a fault.

```
CPU0 (Primary)      ──[Instruction N]──[Instruction N+1]──►
                           │                    │
                      [Compare]            [Compare]
                           │                    │
Shadow Core         ──[Instruction N]──[Instruction N+1]──►
(1 cycle delayed)
                           │ Mismatch detected?
                           ▼
                     Fault → SMU alarm → System reaction
```

### 14.2 Lockstep on TC3xx

On AURIX TC3xx:
- **CPU0 lockstep** is configurable via `BMHD.LOCKSTEP`
- When enabled, a dedicated shadow core (not visible as a programmable CPU) mirrors CPU0 execution
- Lockstep comparison covers register file outputs, memory access addresses and data, and control flow
- A mismatch triggers an SMU (Safety Management Unit) alarm, which can be configured to reset the device, turn on a safe state output, or log the fault

### 14.3 When Lockstep Must Be Active

For ASIL-D systems (the highest automotive safety integrity level), ISO 26262 requires that CPU execution errors be detected with sufficient diagnostic coverage. CPU lockstep provides:
- **Single-point fault detection** — any transient or permanent fault in the primary core is detected
- **Diagnostic coverage** > 99% for CPU execution faults

**Lockstep must be configured before user application code executes.** This is why Boot Firmware configures lockstep as part of the startup sequence — before jumping to STAD.

### 14.4 Lockstep Timing Consideration

During lockstep configuration, there is a brief period where the primary and shadow cores resynchronize. Boot Firmware handles this resynchronization sequence. Application startup code must not disable lockstep after Boot Firmware enables it (doing so would violate the ASIL-D safety case).

---

## 15. Safety Checks During Boot

### 15.1 LBIST — Logic Built-In Self Test

LBIST is a structural test of the CPU core logic gates. On TC3xx, LBIST can be triggered automatically at power-on (depending on device configuration) or on demand via the BIST controller.

LBIST operation:
1. CPU core is taken offline
2. Test patterns are applied to the logic scan chains
3. Signature (a compressed checksum of all scan chain outputs) is compared against a golden reference programmed during manufacturing
4. Match → core passes, returns to operational mode
5. Mismatch → LBIST fault → SMU alarm → system reaction

LBIST detects **permanent manufacturing defects** as well as **latent faults** that have developed during device lifetime (electromigration, hot carrier injection, etc.).

### 15.2 Flash ECC Checks

All accesses to PFlash and DFlash go through hardware ECC logic:
- **SEC-DED (Single Error Correct, Double Error Detect)** Hamming code
- 1-bit errors are **silently corrected** — transparent to software
- 2-bit errors generate a **bus error** — CPU trap
- ECC error status registers (`PFLASH_ECCR`, `DFLASH_ECCR`) log all corrected and uncorrected errors

During boot, reading the BMHD from UCB goes through the same ECC hardware. An ECC error during BMHD read means the BMHD is corrupt — it fails validation regardless of the content.

### 15.3 Watchdog Servicing During Boot

TC3xx has multiple watchdog timers:
- **SCU Watchdog (SCU_WDTS)** — system-wide watchdog, starts running at reset
- **CPU-local watchdogs** — one per CPU core

Boot Firmware must service the system watchdog within the configured timeout window or the device resets. This ensures that if Boot Firmware hangs (e.g., waiting forever for a BMHD that is being read from a faulty flash sector), the watchdog forces a reset and the system recovers.

Startup code must also service or disable watchdogs appropriately — a common startup failure mode is an early watchdog reset because the startup code takes longer than expected (e.g., large SRAM clear taking longer than watchdog timeout).

### 15.4 SMU — Safety Management Unit

The SMU is the central safety monitoring hub of AURIX. It monitors:
- Hardware alarm inputs (ECC errors, voltage monitors, clock monitors, LBIST)
- Software-triggered alarms
- Watchdog timeouts

For each alarm, the SMU can be configured to:
- Trigger a safe state output (e.g., turn off motor drive)
- Generate an NMI to the application
- Trigger a system reset
- Log the alarm for post-boot analysis

During boot, SMU alarms that occur before the application has configured its SMU reaction handlers are handled by **Boot Firmware's default SMU reaction** — typically a system reset.

### 15.5 Checklist: Safety Checks in Boot Order

| Order | Check | Mechanism | Triggers |
|-------|-------|-----------|---------|
| 1 | EVR voltage settling | Hardware comparator | Hard reset if supply out of range |
| 2 | LBIST (if configured) | BIST controller | SMU alarm if signature mismatch |
| 3 | Flash ECC on UCB read | Flash ECC hardware | Bus error / CRC failure if corrupt |
| 4 | BMHD CRC validation | Software in Boot Firmware | Fall through to next BMHD |
| 5 | BMHD inverted CRC | Software in Boot Firmware | Fall through to next BMHD |
| 6 | HSM firmware authentication | HSM ROM hardware | HSM stays in error state |
| 7 | Lockstep synchronization | Lockstep hardware | SMU alarm if compare fails |
| 8 | Watchdog servicing | SCU watchdog timer | System reset if timeout |
| 9 | ABM CRC (if ABM mode) | Software in Boot Firmware | Revert to internal flash boot |
| 10 | Flash ECC during code fetch | Flash ECC hardware | Bus error trap if uncorrectable |

---

## 16. Application Startup Software

### 16.1 What Startup Software Is

Startup software (sometimes called crt0, startup.s, or the C runtime initialization code) is user-provided code that bridges Boot Firmware and the C/C++ application. It begins executing at the STAD address and performs all initialization that must occur before `main()` is called.

On a bare-metal AURIX system, startup software is typically provided by the compiler toolchain (Tasking, GCC/HighTec, GreenHills) or the MCAL/BSW layer (AUTOSAR). It must be customized for the specific TC3xx derivative and memory map.

### 16.2 Mandatory Initialization Steps

The following must be completed before `main()` is called:

#### Step 1: PSW Initialization

The **Program Status Word** is the primary CPU control register. Boot Firmware initializes it to a safe default, but startup code should reconfigure it for the application's needs:

```c
// Enable interrupts, set protection mode, configure IS (interrupt stack pointer)
// PSW.IS = 1 (use interrupt stack pointer for interrupts)
// PSW.CDC = 0 (clear Call Depth Counter)
// PSW.CDE = 1 (enable Call Depth Count)
// PSW.IO = 0b10 (User-1 mode — access to some privileged instructions)
```

#### Step 2: CSA Initialization

The Context Save Area linked list must be constructed in DSPR before any function call, interrupt, or trap can complete:

```c
void __init_CSA(uint32_t* csa_base, uint32_t csa_size) {
    // Divide the CSA region into 64-byte context frames
    // Link them into a free list using hardware-defined format
    // Write head of free list to FCX register
    // Write end of free list to LCX register (limit pointer)
}
```

Failure to initialize the CSA before the first function call or interrupt results in a CSA overflow trap (Class 9 trap) immediately.

#### Step 3: Stack Initialization

CPU0 needs at minimum two stacks:
- **User stack (A10 stack pointer)** — used for function calls and local variables
- **Interrupt stack (IS)** — dedicated stack for interrupt handlers (prevents nested interrupt stack growth into user stack)

```
/* Typical linker symbols */
__CPU0_ISP = start of interrupt stack region + stack size;
__CPU0_USP = start of user stack region + stack size;
```

#### Step 4: .bss Section Clear

All uninitialized global and static variables (`int x;` at file scope) are in the `.bss` section. The C standard guarantees they are zero-initialized. Startup code must zero this memory:

```c
void __init_bss(uint32_t* bss_start, uint32_t* bss_end) {
    while (bss_start < bss_end) {
        *bss_start++ = 0;
    }
}
```

**Watchdog consideration:** If `.bss` is large (e.g., 256 KB of SRAM), this loop takes measurable time. Startup code must service the watchdog during this loop or temporarily disable it.

#### Step 5: .data Section Copy

Initialized global variables (`int x = 42;`) have their initial values stored in a **load memory address (LMA)** in PFlash and must be copied to their **virtual memory address (VMA)** in RAM at startup:

```c
void __init_data(uint32_t* src, uint32_t* dst, uint32_t* dst_end) {
    while (dst < dst_end) {
        *dst++ = *src++;
    }
}
```

#### Step 6: PLL / Clock Initialization

The system clock configuration is user responsibility after Boot Firmware. Boot Firmware runs from the **backup (internal oscillator) clock** — typically 100 MHz maximum on TC3xx. Application startup must:

1. Configure the PLL input source (XTAL or internal oscillator)
2. Program PLL multiplier/divider to achieve the target system frequency
3. Wait for PLL to lock
4. Switch clock source from backup clock to PLL
5. Configure peripheral clock dividers (GTM, GETH, etc.)

**Caution:** Switching the clock source while executing from the same clock is a critical operation. Startup code must follow the exact clock switch sequence in the TC3xx User Manual to avoid a momentary clock glitch that could corrupt ongoing operations.

#### Step 7: Interrupt Vector Table

The **Interrupt Vector Table (BTV — Base Trap Vector / BIV — Base Interrupt Vector)** registers must be set to point to the application's trap and interrupt handler tables:

```c
// BIV: Base Interrupt Vector register
// Points to the 32-entry interrupt service routine vector table
// Each entry is 8 bytes, containing the ISR address and CSA context info
__mtcr(CPU_BIV, (uint32_t)__interrupt_vector_table);

// BTV: Base Trap Vector register
// Points to the 8-entry trap handler vector table
__mtcr(CPU_BTV, (uint32_t)__trap_vector_table);
```

If BIV/BTV are not set, any interrupt or trap will jump to the Boot Firmware's (now-finished) vector table — causing an immediate crash.

#### Step 8: Peripheral Initialization

Depending on the system, startup code may initialize:
- GPIO pin assignments
- CAN/LIN/Ethernet PHY reset sequences
- External memory controllers (EMEM)
- Power management sequences

For AUTOSAR-based systems, this is handled by the BSW (Basic Software) initialization sequence and EcuM (ECU State Manager).

### 16.3 C Runtime Initialization vs. Application Init

Startup code should be split into two phases:

| Phase | What | When |
|-------|------|------|
| C runtime init | CSA, stack, .bss, .data, BTV, BIV, clock | Before `main()` |
| Application init | RTOS init, AUTOSAR BSW init, peripheral drivers | Inside `main()` / OS task |

Mixing peripheral init into the C runtime startup phase is a common pitfall — if a fault occurs during peripheral init, there is no fault handler yet (BIV not set), making debugging extremely difficult.

### 16.4 Startup Code Entry Point

The startup code entry point is placed at the STAD address in the linker script:

```ld
/* Example TC3xx linker script snippet */
MEMORY {
    PFlash_0 (rx) : ORIGIN = 0x80000000, LENGTH = 2M
}

SECTIONS {
    .startup : {
        KEEP(*(.startup))      /* startup code entry at STAD */
    } > PFlash_0
    
    .text : {
        *(.text*)
    } > PFlash_0
}
```

The startup section must be the very first code in PFlash Bank 0, and its address must match the `STAD` field programmed in BMHD.

---

## 17. Multicore Startup

### 17.1 The Multicore Boot Contract

The TC3xx multicore startup follows a strict contract:

1. CPU0 completes **all shared resource initialization** before releasing any other core.
2. Each released core performs its own **private initialization** (own stack, own CSA, own BIV/BTV).
3. Each core then starts its application task or OS scheduler.

Violating the contract — for example, releasing CPU1 before SRAM is initialized, or before the PLL is configured — leads to non-deterministic behavior that is difficult to debug and impossible to certify.

### 17.2 Core Release Mechanism

CPU0 releases another core by:

1. Writing the **start address** of the other core's startup code to the core's `CORE_SEL` register
2. Writing a specific **start trigger** value to `CPUx_PC` (program counter set) register
3. Clearing the **`BHALT`** bit in the `CPUx_DBGSR` register

```c
/* Release CPU1 to start at cpu1_start_address */
CPU1_PC.U     = (uint32_t)&cpu1_startup;   /* Set start address */
CPU1_DBGSR.B.BHALT = 0;                    /* Release from halt */
```

After this sequence, CPU1 begins fetching and executing instructions from `cpu1_startup`.

### 17.3 Startup Synchronization

Because CPU0 releases other cores asynchronously, synchronization barriers are needed to ensure that shared initialization is complete before dependent cores proceed:

```c
/* CPU0 side */
shared_init_flag = 0;
init_shared_memory();
init_gtm();
init_can_driver();
shared_init_flag = 1;   /* Signal: shared init done */

release_cpu1();
release_cpu2();

/* CPU1 side (runs after CPU1 startup code completes) */
while (shared_init_flag == 0) {
    /* Spin-wait */
}
/* Now safe to use shared resources */
start_cpu1_tasks();
```

The shared flag must be in a memory region accessible by both cores — typically LMU (Local Memory Unit / Global RAM).

### 17.4 Per-Core Startup Requirements

Each released core must initialize its own:

| Resource | Notes |
|----------|-------|
| CSA free list | Each core has its own CSA region in its own DSPR |
| Stack pointer (A10) | Each core has its own stack in its own DSPR |
| Interrupt stack pointer | Separate interrupt stack per core |
| BIV register | Can point to shared or per-core interrupt table |
| BTV register | Usually shared trap vector table |
| Local watchdog | Each core has its own CPU watchdog |
| PCXI register | Previous context register — must be valid |

### 17.5 AUTOSAR Multicore

In an AUTOSAR OS environment, multicore startup follows the AUTOSAR OS specification:
- CPU0 runs `StartOS()` which initializes the OS
- The OS activates the startup hooks on other cores
- `StartupHook()` on each core handles per-core hardware init
- The OS scheduler takes over after all cores complete `StartOS()`

---

## 18. Watchdog Management During Boot

### 18.1 TC3xx Watchdog Architecture

TC3xx has multiple watchdog timers:

| Watchdog | Scope | Timeout type |
|----------|-------|--------------|
| SCU system watchdog (`SCU_WDTS`) | System-wide | Timer window (open/close) |
| CPU0 watchdog (`SCU_WDTCPU0`) | CPU0 only | Timer window |
| CPU1 watchdog (`SCU_WDTCPU1`) | CPU1 only | Timer window |
| CPUn watchdog (`SCU_WDTCPUn`) | CPUn only | Timer window |

The **window watchdog** model means the watchdog must be serviced within a **specific time window** — not too early, not too late. This is stricter than a simple "service before timeout" watchdog. Early service (before the window opens) is also an error, which catches runaway loops that service the watchdog too frequently.

### 18.2 Watchdog Password Protection

TC3xx watchdogs are **password-protected** — a service sequence requires writing the correct password before the watchdog accepts the service:

```c
/* Unlock (password phase) */
SCU_WDTS_CON0.U = ((SCU_WDTS_CON0.U ^ 0xFFu) & ~(1u << 1u)) | (1u << 0u);
/* Modify (modify phase) */
SCU_WDTS_CON0.U = ((SCU_WDTS_CON0.U ^ 0xFFu) & ~(1u << 0u)) | (1u << 1u);
```

An incorrect password write triggers an immediate watchdog reset. This prevents runaway software from accidentally servicing the watchdog.

### 18.3 Boot Firmware Watchdog Handling

Boot Firmware services the system watchdog during its initialization sequence. If Boot Firmware execution takes longer than expected (e.g., reading from a slow or faulty flash sector), the watchdog fires and forces a system reset — a safe recovery behavior.

### 18.4 Startup Code Watchdog Recommendations

1. **Do not disable the watchdog permanently** during startup — reconfigure its timeout to a value compatible with startup time.
2. **Service the watchdog** during long initialization loops (`.bss` clear, SRAM test, large array initialization).
3. **Configure per-CPU watchdogs** as part of each CPU's startup sequence before entering the OS.
4. **Use AUTOSAR WdgM** (Watchdog Manager) in AUTOSAR systems to coordinate multi-CPU watchdog servicing.

---

## 19. Non-Volatile Memory Layout

### 19.1 Program Flash (PFlash) Organization

```
PFlash Bank 0 (typical TC39x — 8 MB)
┌────────────────────────────────────────────────────┐
│ 0x80000000  Startup code (STAD entry point)        │
│ 0x80000020  ...                                    │
│ ...         Application code                       │
│ 0x807FFFFF  End of Bank 0                          │
└────────────────────────────────────────────────────┘

PFlash Bank 1 (typical TC39x — 8 MB)
┌────────────────────────────────────────────────────┐
│ 0x80800000  [ABM region — alternate boot image]    │
│             ABM header at start of region          │
│             Alternate application code             │
│ 0x80FFFFFF  End of Bank 1                          │
└────────────────────────────────────────────────────┘
```

### 19.2 Data Flash (DFlash) / UCB Organization

```
Data Flash / UCB Area
┌──────────────────────────────────────────────────────┐
│  UCB_BMHD0_ORIG   (Boot Mode Header 0, original)     │
│  UCB_BMHD0_COPY   (Boot Mode Header 0, copy)         │
│  UCB_BMHD1_ORIG                                      │
│  UCB_BMHD1_COPY                                      │
│  UCB_BMHD2_ORIG                                      │
│  UCB_BMHD2_COPY                                      │
│  UCB_BMHD3_ORIG                                      │
│  UCB_BMHD3_COPY                                      │
│  UCB_PFLASH       (PFlash protection configuration)  │
│  UCB_DFLASH       (DFlash protection configuration)  │
│  UCB_DBG          (Debug protection settings)        │
│  UCB_HSM          (HSM configuration)                │
│  UCB_IFX          (Infineon tuning — read-only)      │
│  EEPROM emulation sectors (user Data Flash)          │
└──────────────────────────────────────────────────────┘
```

### 19.3 Flash Sector Sizes

| Region | Erase unit | Program unit |
|--------|-----------|--------------|
| PFlash | 16 KB (small sectors near start) / 64 KB (large sectors) | 32 bytes (burst) |
| DFlash / UCB | 4 KB | 8 bytes |
| HSM Flash | Managed by HSM | N/A for TriCore |

---

## 20. Complete Boot Sequence Diagram

```
Power Applied / PORST
         │
         ▼
   EVR Stabilization
   (1.3 V + 3.3 V regulators settle)
         │
         ▼
   ┌─────────────┐
   │  Boot ROM   │  ── Infineon-programmed, immutable
   │  (BROM)     │  ── Minimal HW init, Flash startup
   └─────────────┘
         │
         ▼
   ┌───────────────────────────────────────┐
   │         Boot Firmware                 │
   │                                       │
   │  1. Read SCU_RSTSTAT                  │
   │  2. Read BMHD0..3 from UCB            │
   │  3. Validate BMHD (ID + CRC + INVCRC) │
   │  4. Start HSM (if BMHD.HSM = 1)      │
   │  5. Configure lockstep               │
   │  6. Evaluate BOOTMODE field          │
   └───────────────────────────────────────┘
         │
         ├─────────────────┬──────────────────────────┐
         │                 │                          │
         ▼                 ▼                          ▼
   BOOTMODE = 00     BOOTMODE = 01             BOOTMODE = 10/11
   Internal Flash    Alternate Boot Mode       Bootstrap Loader
         │                 │                          │
         │           Verify ABM header           Init ASC/CAN
         │           Verify app CRC              Receive code
         │                 │                    Write to PSPR
         │           PASS  │  FAIL                    │
         │             ▼   │   ▼                      │
         │          Jump to  Fallback              Execute
         │          ABMSTAD  to flash                PSPR
         │
         ▼
   ┌──────────────────────────────────────────────┐
   │   Application Startup Code (at STAD)         │
   │                                              │
   │   1. PSW initialization                      │
   │   2. CSA free list construction              │
   │   3. Stack pointer init (user + interrupt)   │
   │   4. .bss section zero-initialization        │
   │   5. .data section copy (PFlash → SRAM)      │
   │   6. PLL / clock configuration               │
   │   7. BIV / BTV (interrupt/trap vectors)      │
   │   8. Peripheral initialization               │
   └──────────────────────────────────────────────┘
         │
         ▼
   ┌──────────────────────────────────────────────┐
   │   Multicore Release                          │
   │                                              │
   │   CPU0 writes start addresses:               │
   │     CPU1_PC = &cpu1_startup                  │
   │     CPU2_PC = &cpu2_startup                  │
   │     CPU3_PC = &cpu3_startup  (if present)    │
   │   CPU0 clears BHALT for CPU1..N              │
   └──────────────────────────────────────────────┘
         │
         ▼ (each CPU in parallel)
   ┌──────────────────────────────────────────────┐
   │   Per-Core Startup (CPU1, CPU2, ...)         │
   │   (same steps as CPU0 startup, private data) │
   └──────────────────────────────────────────────┘
         │
         ▼
   ┌──────────────────────────────────────────────┐
   │   main() / RTOS Scheduler / AUTOSAR OS       │
   │                                              │
   │   System fully operational                   │
   └──────────────────────────────────────────────┘
```

---

## 21. Common Boot Failures and Debugging

### 21.1 BMHD Validation Failure

**Symptom:** Device falls into BSL mode unexpectedly or fails to boot.

**Causes:**
- BMHD was never programmed (blank device)
- UCB write was interrupted (power loss during UCB programming) — both ORIG and COPY corrupted
- Incorrect CRC field computed by programmer tool
- Wrong BMHDID magic value (off-by-one error in flash writer)

**Debug approach:**
1. Use AURIX Development Studio OCDS debugger to halt at BROM entry and single-step through Boot Firmware
2. Use a flash programmer (Infineon MemTool, TRACE32) to read back and inspect UCB_BMHD0_ORIG and UCB_BMHD0_COPY
3. Verify BMHDID field equals `0xB359`
4. Manually compute CRC over BMHD structure and compare against stored `CRCBMHD`

### 21.2 Startup Crash Before main()

**Symptom:** Device resets repeatedly, never reaches `main()`.

**Causes:**
- CSA not initialized before first function call → Class 9 trap (CSA overflow)
- Stack pointer not set → stack underflow/overflow immediately
- .data copy source/destination mismatch → corrupts RAM on startup
- BTV not set → any trap jumps to garbage address → CPU trap cascade
- Watchdog fires during long initialization

**Debug approach:**
1. Connect OCDS debugger, set breakpoint at STAD entry point
2. Single-step through startup code checking register values after each section
3. Monitor `SCU_RSTSTAT.TP` (trap reset) bit — if set, a trap caused the reset
4. Read `CPU0_PCXI` to find the trap class
5. Check `SCU_WDTS_SR.TIM` to see if watchdog counter is near zero

### 21.3 Lockstep Fault at Boot

**Symptom:** SMU alarm shortly after startup, system resets immediately after application starts.

**Causes:**
- Lockstep enabled in BMHD but startup code intentionally disables it → SMU alarm on comparison failure
- Hardware defect in shadow core (extremely rare — indicates physical damage or manufacturing defect)
- Application code writing to CPU control registers that affect lockstep (e.g., incorrect inline assembly)

**Debug approach:**
1. Read `SMU_AD0..2` (Alarm Debug registers) to identify which alarm fired
2. Check `CPU0_DLMU_SPROT_ACCENA` — spurious writes to protected registers
3. Verify startup code does not touch lockstep configuration registers

### 21.4 HSM Failing to Initialize

**Symptom:** Boot halts waiting for HSM ready signal; or HSM reports authentication failure.

**Causes:**
- HSM firmware in HSM Flash corrupted (ECC error or bad OTA update)
- Key store empty — device shipped without provisioning
- OEM public key mismatch — application was signed with wrong key
- HSM firmware version incompatible with TriCore application

**Debug approach:**
1. Check `HSM_FSTAS` (HSM firmware status register, accessible from TriCore side)
2. Review HSM error codes via the mailbox status registers
3. Verify HSM Flash contents using Infineon HSM debug tools (restricted access — requires NDA)
4. Validate OEM public key provisioning using a known-good test device

### 21.5 Incorrect STAD Address

**Symptom:** CPU immediately takes a bus error trap or alignment trap after Boot Firmware hands off.

**Causes:**
- STAD points to an erased PFlash region (all 0xFF or 0x00)
- STAD misaligned (not 4-byte aligned)
- STAD set to data section address instead of code section address
- Linker script and BMHD STAD field out of sync

**Debug approach:**
1. Halt at BROM with OCDS debugger before STAD jump
2. Read `BMHD.STAD` value and check it matches the linker map's startup code address
3. Read flash contents at STAD address — should contain valid TriCore instructions (not 0xFFFFFFFF)

---

## 22. AURIX Development Studio and Boot Debugging

### 22.1 OCDS — On-Chip Debug Support

AURIX TC3xx implements **OCDS** (On-Chip Debug Support), Infineon's proprietary debug architecture. It provides:
- JTAG and DAP (Debug Access Port) interfaces
- Breakpoints (code and data)
- Single-step execution
- Register and memory read/write
- Real-time trace (Nexus / Aurora interfaces on some derivatives)

OCDS connects to all CPU cores simultaneously, allowing cross-core breakpoints and synchronized halt/resume.

### 22.2 Debugging Early Boot

To debug Boot Firmware and startup code:

1. **Connect before reset** — use the debugger's "connect and halt before BROM" option
2. **Set a software breakpoint at STAD** — the debugger will run through BROM and Boot Firmware transparently and halt at the first user instruction
3. **Inspect BMHD readout** — after Boot Firmware runs, use memory window to inspect the BMHD structure that was loaded
4. **Check reset cause** — read `SCU_RSTSTAT` in the register window to understand which reset type triggered

### 22.3 MemTool for Flash Programming

Infineon's **MemTool** (and command-line variant `UDE_STK`) is the standard tool for programming TC3xx devices:
- Programs PFlash with application binary
- Programs UCB/BMHD with configuration
- Can selectively erase sectors
- Supports Intel HEX, Motorola S-Record, and ELF formats

For production, **TRACE32** from Lauterbach or **ULINKpro** from Keil/ARM are commonly used for high-speed gang programming.

---

## 23. Summary and Key Principles

### 23.1 The Boot Sequence in One Paragraph

After any reset, CPU0 enters the Infineon-programmed Boot ROM, which stabilizes the hardware and starts Boot Firmware. Boot Firmware reads the Boot Mode Header from UCB Data Flash, validates it through a 4-stage CRC check, starts the HSM if configured, enables CPU0 lockstep, and jumps to the application start address (STAD) in Program Flash. Application startup code initializes the C runtime environment — CSA, stacks, .bss, .data, clock, interrupt vectors — and then releases other CPU cores. Each core performs its own private initialization. Once all cores are running, the RTOS or application scheduler takes over and the system is fully operational.

### 23.2 Seven Non-Negotiable Rules for TC3xx Boot

1. **CPU0 is the unconditional boot master.** No other core executes before CPU0 explicitly releases it.

2. **Boot Firmware runs before any user code.** It cannot be bypassed, patched, or instrumented from user software.

3. **BMHD is the single source of truth for boot configuration.** All boot mode decisions flow from a validated BMHD.

4. **Safety features (lockstep, HSM) are configured before application code executes.** Never disable or reconfigure them in application code without a certified safety analysis.

5. **Startup code must initialize CSA before any function call.** A missing CSA initialization is the most common cause of mysterious early-boot crashes.

6. **The watchdog runs from reset.** Startup code must service it — the watchdog does not know that your device is still initializing.

7. **HSM is not optional in production automotive systems.** A TC3xx without HSM-based secure boot is a security vulnerability waiting to be exploited.

### 23.3 Boot is a Safety and Security Gate

The AURIX boot process is explicitly designed to be a **safety and security gateway** before the application runs. It ensures that:
- Only authenticated firmware executes (HSM secure boot)
- CPU hardware is defect-free (LBIST)
- CPU execution is monitored for faults (lockstep)
- Configuration is intact and uncorrupted (BMHD CRC)
- Hardware is in a known-good state (EVR, flash ECC)

Any attempt to shortcut or bypass these checks — common in development for convenience — should be treated as a **deliberate safety and security regression** that requires formal justification before shipping to production.

---

## 24. Glossary

| Term | Definition |
|------|-----------|
| **ABM** | Alternate Boot Mode — secondary application image path with independent CRC verification |
| **ASIL** | Automotive Safety Integrity Level — ISO 26262 severity classification (A to D, D highest) |
| **BIV** | Base Interrupt Vector — CPU register pointing to the interrupt handler table |
| **BMHD** | Boot Mode Header — UCB data structure controlling TC3xx startup behavior |
| **BROM** | Boot ROM — Infineon-programmed immutable code executed first after reset |
| **BSL** | Bootstrap Loader — UART/CAN-based code download interface for device programming |
| **BTV** | Base Trap Vector — CPU register pointing to the trap handler table |
| **CSA** | Context Save Area — linked list of memory frames for hardware interrupt/trap context storage |
| **DFlash** | Data Flash — NVM region for data and UCB configuration (distinct from PFlash) |
| **DSPR** | Data Scratchpad RAM — per-CPU private data RAM |
| **ECC** | Error Correcting Code — hardware mechanism to detect and correct bit errors in memory |
| **EVITA** | E-safety Vehicle Intrusion Protected Applications — European security specification for automotive HSMs |
| **FPU** | Floating-Point Unit — hardware floating-point execution engine |
| **HSM** | Hardware Security Module — isolated security processor for cryptography and key management |
| **LBIST** | Logic Built-In Self Test — structural test of CPU core gate-level logic |
| **LMU** | Local Memory Unit — global RAM accessible by all cores without SRI arbitration |
| **OCDS** | On-Chip Debug Support — Infineon's JTAG/DAP-based debug infrastructure |
| **OTA** | Over-The-Air — wireless firmware update mechanism |
| **PFlash** | Program Flash — NVM region for application code |
| **PORST** | Power-On Reset — cold start reset triggered by power supply application |
| **PSP** | Program Scratchpad RAM (same as PSPR) |
| **PSPR** | Program Scratchpad RAM — per-CPU private instruction RAM; BSL download target |
| **PSW** | Program Status Word — primary CPU control and status register |
| **SMU** | Safety Management Unit — central safety alarm and reaction coordinator |
| **STAD** | Start Address — BMHD field specifying the application entry point in PFlash |
| **TRNG** | True Random Number Generator — entropy source for cryptographic operations |
| **UCB** | User Configuration Block — protected DFlash region storing BMHD and device configuration |

---

## 25. References

1. **Infineon Technologies — AURIX TC3xx User Manual**  
   Part number: `TC3xx_UM_vX.Y` — The primary reference for all register descriptions, memory maps, and functional descriptions. Available via myInfineon portal (registration required for confidential sections).

2. **Infineon AP32381 — AURIX TC3xx Startup and Initialization Application Note**  
   Available at: `documentation.infineon.com/aurixtc3xx` — Covers the official startup sequence for TC3xx including PSW initialization and reset evaluation.

3. **Infineon AURIX TC3xx Safety Manual**  
   Version 2.0, 2021-05-03 — Documents ISO 26262 ASIL-D safety mechanisms including LBIST, lockstep, SMU, and ECC.

4. **EVITA D3.3 — Hardware Security Module Specification**  
   EVITA Project (E-safety Vehicle Intrusion Protected Applications) — Defines Full/Medium/Light HSM requirements referenced by TC3xx HSM design.

5. **ISO 26262:2018 — Road Vehicles — Functional Safety**  
   Part 5 (Product Development at Hardware Level) is particularly relevant for LBIST, lockstep, and hardware safety mechanisms discussed in this document.

6. **ISO/SAE 21434:2021 — Road Vehicles — Cybersecurity Engineering**  
   Relevant for HSM secure boot, key management, and OTA update security requirements.

7. **UN Regulation No. 155 — Cyber Security and Cyber Security Management System**  
   Mandatory EU regulation for vehicle type approval — drives HSM requirements in production TC3xx designs.

8. **Infineon AURIX TC3xx Platform Family User Manual (1881 pages)**  
   Available at: `manuals.plus` and via `documentation.infineon.com/aurixtc3xx` — Comprehensive hardware reference.

9. **AURIX Development Studio Documentation**  
   `https://www.infineon.com/aurixdevelopmentstudio` — Tool chain documentation for eclipse-based IDE, flash programming, and OCDS debug.

10. **TriCore 1.6.2P Architecture Volume 1 & 2**  
    Infineon TriCore architecture manuals covering ISA, pipeline, CSA mechanism, and memory protection.

---

*Document generated for engineering reference. All register names, bit fields, and address values should be verified against the device-specific User Manual appendix for the exact TC3xx derivative in use (TC36x / TC37x / TC38x / TC39x). Some fields vary between derivatives.*