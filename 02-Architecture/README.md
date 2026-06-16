# TriCore Architecture Deep Dive
### Part 2 — Core Architecture, CPU Features, Safety & Performance

> **TriCore** is not a standard embedded CPU. It is a purpose-built automotive processor
> that merges real-time control, DSP mathematics, and RISC efficiency into one unified
> architecture — designed from day one to run safely inside a moving vehicle.

---

## Table of Contents

| # | Topic |
|---|---|
| 1 | [Introduction](#1--introduction) |
| 2 | [TriCore Design Philosophy](#2--tricore-design-philosophy) |
| 3 | [Why TriCore is Different](#3--why-tricore-is-different) |
| 4 | [CPU Core Overview](#4--cpu-core-overview) |
| 5 | [Harvard Architecture](#5--harvard-architecture) |
| 6 | [RISC Architecture](#6--risc-architecture) |
| 7 | [Register Architecture](#7--register-architecture) |
| 8 | [Instruction Set Architecture](#8--instruction-set-architecture) |
| 9 | [16-bit and 32-bit Instructions](#9--16-bit-and-32-bit-instructions) |
| 10 | [DSP Engine](#10--dsp-engine) |
| 11 | [SIMD Operations](#11--simd-operations) |
| 12 | [Multiply-Accumulate (MAC)](#12--multiply-accumulate-mac-operations) |
| 13 | [Floating Point Unit (FPU)](#13--floating-point-unit-fpu) |
| 14 | [Pipeline Architecture](#14--pipeline-architecture) |
| 15 | [Superscalar Execution](#15--superscalar-execution) |
| 16 | [Branch Prediction](#16--branch-prediction) |
| 17 | [Memory Architecture](#17--memory-architecture) |
| 18 | [Cache Architecture](#18--cache-architecture) |
| 19 | [Memory Protection Unit (MPU)](#19--memory-protection-unit-mpu) |
| 20 | [Privilege Levels](#20--privilege-levels) |
| 21 | [Interrupt System Overview](#21--interrupt-system-overview) |
| 22 | [Trap System Overview](#22--trap-system-overview) |
| 23 | [Multi-Core Processing](#23--multi-core-processing) |
| 24 | [Inter-Core Communication](#24--inter-core-communication) |
| 25 | [Lockstep Safety](#25--lockstep-safety) |
| 26 | [Functional Safety Features](#26--functional-safety-features) |
| 27 | [Security Features](#27--security-features) |
| 28 | [Watchdogs](#28--watchdogs) |
| 29 | [Communication Capabilities](#29--communication-capabilities) |
| 30 | [Automotive Advantages](#30--automotive-advantages) |
| 31 | [TriCore vs ARM Cortex-M](#31--tricore-vs-arm-cortex-m) |
| 32 | [Summary](#32--summary) |

---

## 1 · Introduction

TriCore is Infineon's proprietary processor architecture — the CPU heart of every AURIX
microcontroller. It was not adapted from a general-purpose design; it was built
specifically for automotive embedded systems where the following properties are
**non-negotiable**:

| Property | What It Means in a Vehicle |
|---|---|
| **High computational performance** | Motor control, sensor fusion, radar need heavy math |
| **Deterministic real-time behavior** | A brake command must respond in microseconds, every time |
| **DSP processing capability** | Signal filtering and motor FOC algorithms need MAC hardware |
| **Functional safety** | A CPU bug must never silently command the wrong torque |
| **Automotive-grade reliability** | Operates from −40 °C to +150 °C across 15+ years |

---

## 2 · TriCore Design Philosophy

TriCore unifies three historically separate processor concepts into a **single CPU core**:

```
                    ┌──────────────────────────┐
                    │         TriCore          │
                    │      CPU Architecture    │
                    └────────────┬─────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                   │                    │
            ▼                   ▼                    ▼
   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
   │      MCU        │ │      DSP        │ │      RISC       │
   │                 │ │                 │ │                 │
   │ Real-time I/O   │ │ Fast math       │ │ High-throughput │
   │ Interrupt ctrl  │ │ MAC operations  │ │ instruction     │
   │ Peripheral mgmt │ │ SIMD, filtering │ │ execution       │
   │ Timer control   │ │ Motor control   │ │ Compiler-       │
   │                 │ │ algorithms      │ │ friendly ISA    │
   └─────────────────┘ └─────────────────┘ └─────────────────┘
```

### The Three Design Goals

```
  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
  │  HIGH PERFORMANCE│   │REAL-TIME DETERMIN│   │ FUNCTIONAL SAFETY│
  │                  │   │     ISM          │   │                  │
  │  Process-heavy   │   │  Every deadline  │   │  Every fault     │
  │  automotive      │   │  must be met     │   │  must be caught  │
  │  algorithms at   │   │  with guaranteed │   │  before it       │
  │  MCU convenience │   │  worst-case      │   │  becomes a       │
  │                  │   │  timing          │   │  danger          │
  └──────────────────┘   └──────────────────┘   └──────────────────┘
```

---

## 3 · Why TriCore is Different

### What a Generic Embedded Processor Covers

```
  GPIO control
  Basic timers
  UART / SPI / I2C
  Simple ADC reads
```

### What Automotive Systems Actually Demand

```
  ┌──────────────────────────────────────────────────────┐
  │           Automotive Processing Demands              │
  │                                                      │
  │  ⚡ EV Motor Control    → FOC algorithm, PWM, ADC    │
  │  📡 Radar Processing    → FFT, CFAR, SIMD math       │
  │  🔋 Battery Management  → Coulomb counting, models   │
  │  🛡️  Safety Monitoring   → Lockstep, ECC, watchdogs  │
  │  🌐 Vehicle Networking  → CAN FD, Ethernet, FlexRay  │
  │  🤖 Sensor Fusion       → Kalman filters, matrix ops │
  └──────────────────────────────────────────────────────┘
```

A standard Cortex-M cannot cover all of these at the required performance and safety
levels. TriCore was built precisely for this gap.

---

## 4 · CPU Core Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                    TriCore CPU Core                      │
  │                                                          │
  │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   │
  │   │  Register   │   │    ALU      │   │  DSP Unit   │   │
  │   │    File     │   │             │   │             │   │
  │   │  D0–D15     │   │ Arithmetic  │   │ MAC, SIMD   │   │
  │   │  A0–A15     │   │ Logic ops   │   │ Saturation  │   │
  │   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   │
  │          │                 │                  │          │
  │          └─────────────────┼──────────────────┘          │
  │                            │                             │
  │                     ┌──────▼──────┐                      │
  │                     │   Pipeline  │                      │
  │                     │  (5-stage)  │                      │
  │                     └──────┬──────┘                      │
  │                            │                             │
  │                     ┌──────▼──────┐                      │
  │                     │ Memory Sys  │                      │
  │                     │ Cache + MPU │                      │
  │                     └─────────────┘                      │
  └──────────────────────────────────────────────────────────┘
```

The CPU is optimized for **both control workloads** (interrupt handling, I/O management)
and **computational workloads** (signal processing, motor control math).

---

## 5 · Harvard Architecture

TriCore uses a **modified Harvard architecture**, meaning instruction fetch and data
access use **physically separate buses**.

```
  ┌──────────────────┐          ┌──────────────────┐
  │  Instruction     │          │   Data Memory    │
  │  Memory (Flash)  │          │   (SRAM, PFLASH) │
  └────────┬─────────┘          └────────┬─────────┘
           │                             │
           │  Instruction bus            │  Data bus
           │                             │
           └──────────────┬──────────────┘
                          │
                   ┌──────▼──────┐
                   │ TriCore CPU │
                   └─────────────┘
```

### Why This Matters

| Von Neumann | Harvard (TriCore) |
|---|---|
| One bus for instructions + data | Separate buses for each |
| Instructions and data compete for bandwidth | Simultaneous fetch + data access |
| Lower peak throughput | Higher throughput, no bottleneck |

> The CPU can **fetch the next instruction** and **read/write data** for the current
> instruction at the **same time** — eliminating memory bus contention.

---

## 6 · RISC Architecture

TriCore follows RISC (Reduced Instruction Set Computer) principles:

```
  RISC Design Principles in TriCore
  ──────────────────────────────────────────────────────────
  ✓ Simple, uniform instruction formats
  ✓ Load/store architecture (math on registers, not memory)
  ✓ Fixed and predictable execution timing
  ✓ Efficient pipelining (one instruction per stage)
  ✓ Compiler-friendly register model
```

### Benefit for Automotive Systems

```
  Predictable instruction timing
           │
           ▼
  Predictable interrupt latency
           │
           ▼
  Deterministic real-time behavior
           │
           ▼
  Safety requirements can be verified
```

---

## 7 · Register Architecture

TriCore uses **two completely separate register files** — a deliberate architectural
choice that enables better parallelism and compiler optimization.

### Data Registers — `D0` to `D15`

```
  ┌────┬────┬────┬────┬────┬────┬────┬────┐
  │ D0 │ D1 │ D2 │ D3 │ D4 │ D5 │ D6 │ D7 │  32-bit general purpose
  ├────┼────┼────┼────┼────┼────┼────┼────┤
  │ D8 │ D9 │D10 │D11 │D12 │D13 │D14 │D15 │  data registers
  └────┴────┴────┴────┴────┴────┴────┴────┘

  Purpose: Arithmetic · Logic · DSP calculations · Return values
```

### Address Registers — `A0` to `A15`

```
  ┌────┬────┬────┬────┬────┬────┬────┬────┐
  │ A0 │ A1 │ A2 │ A3 │ A4 │ A5 │ A6 │ A7 │  32-bit address/pointer
  ├────┼────┼────┼────┼────┼────┼────┼────┤
  │ A8 │ A9 │A10 │A11 │A12 │A13 │A14 │A15 │  registers
  └────┴────┴────┴────┴────┴────┴────┴────┘

  Purpose: Pointers · Memory addressing · Stack (A10=SP) · Return addr (A11)
```

### Special-Purpose Registers

```
  PSW   — Program Status Word (flags, privilege, call depth)
  PC    — Program Counter
  PCXI  — Previous Context Pointer (interrupt/call chain)
  FCX   — Free Context List head
  LCX   — Last Context List entry
```

### Why Separate Data and Address Registers?

```
  Combined register file (ARM Cortex-M)    Separate files (TriCore)
  ────────────────────────────────         ─────────────────────────
  R0–R15 used for both data & addr    →    D0–D15: only data
                                           A0–A15: only addresses

  Consequence:                             Consequence:
  Compiler must carefully allocate    →    Compiler freely allocates
  registers across both uses               each type independently
  → More register spills to stack          → Fewer spills, better code
```

---

## 8 · Instruction Set Architecture

TriCore's ISA provides six instruction categories:

```
  ┌───────────────┬───────────────────────────────────────────────┐
  │ Category      │ Examples                                      │
  ├───────────────┼───────────────────────────────────────────────┤
  │ Arithmetic    │ ADD, SUB, MUL, DIV, ABS, NEG                 │
  │ Logical       │ AND, OR, XOR, NOT, shift, rotate              │
  │ Branch        │ JEQ, JNE, JLT, CALL, RET, JA, LOOP          │
  │ Load / Store  │ LD.W, ST.W, LD.B, ST.B, LD.D (64-bit pair)  │
  │ DSP           │ MADD, MSUB, MUL.H, MULR.H, DVINIT           │
  │ System        │ ENABLE, DISABLE, MTCR, MFCR, SYSCALL, RFE   │
  └───────────────┴───────────────────────────────────────────────┘
```

---

## 9 · 16-bit and 32-bit Instructions

TriCore supports a **mixed-width instruction set** — instructions are either 16 or 32 bits
wide, and the CPU determines the width from the opcode.

```
  32-bit instruction  →  Full operand flexibility
  ┌────────────────────────────────┐
  │  opcode │  reg  │  reg  │ imm  │  = 4 bytes
  └────────────────────────────────┘

  16-bit instruction  →  Common operations, compact encoding
  ┌────────────────────┐
  │  opcode │  reg  │   │  = 2 bytes
  └────────────────────┘
```

### Impact on Code Density

```
  Same program logic:

  32-bit only ISA            Mixed 16/32-bit (TriCore)
  ─────────────────          ──────────────────────────
  100 instructions           ~65 instructions (avg)
  × 4 bytes each             × ~2.6 bytes (mixed)
  = 400 bytes Flash          = ~169 bytes Flash

  Result: TriCore programs use significantly less Flash
  → Enables lower-cost devices and better cache hit rate
```

---

## 10 · DSP Engine

The DSP engine is one of TriCore's **most powerful differentiators** from standard
microcontrollers.

### What the DSP Engine Handles

```
  ┌─────────────────────────────────────────────────────┐
  │                  DSP Engine Usage                   │
  │                                                     │
  │  🔁 Digital Filters    FIR, IIR — audio, sensor     │
  │  ⚡ Motor Control      FOC, SVPWM — EV drivetrains  │
  │  📡 Radar Processing   Range/velocity estimation    │
  │  🔋 BMS Algorithms     SOC/SOH estimation models    │
  │  🤖 Sensor Fusion      Kalman filter updates        │
  └─────────────────────────────────────────────────────┘
```

### Without a DSP Engine

```
  FIR filter, 32 taps, software-only on Cortex-M4:
  ~32 multiplications + 32 additions = 64+ instructions per sample
  → CPU heavily loaded, less time for control tasks
```

### With TriCore DSP Engine

```
  Same FIR filter using MADD instructions:
  → Completed in far fewer cycles
  → CPU freed for other real-time tasks
```

---

## 11 · SIMD Operations

**SIMD** = **S**ingle **I**nstruction, **M**ultiple **D**ata

A single instruction operates on **multiple data elements packed into one register**.

### Concept

```
  Without SIMD (scalar):           With SIMD (packed 16-bit):
  ────────────────────             ──────────────────────────

  ADD  D1, D2   → A₁+B₁           One instruction:
  ADD  D3, D4   → A₂+B₂           PADD.H  D0, D1
  ADD  D5, D6   → A₃+B₃
  ADD  D7, D8   → A₄+B₄           Operates on register packed as:
                                   ┌──────────┬──────────┐
  4 instructions                   │  A₁ (16) │  A₂ (16) │  D0
                                   └──────────┴──────────┘
                                   ┌──────────┬──────────┐
                                        +          +
                                   ┌──────────┬──────────┐
                                   │  B₁ (16) │  B₂ (16) │  D1
                                   └──────────┴──────────┘
                                   = 2 parallel additions
                                     in 1 instruction
```

### Automotive Use Case

Signal processing routines (e.g., phase current sampling in motor control) benefit
directly from SIMD — multiple channel samples processed per instruction.

---

## 12 · Multiply-Accumulate (MAC) Operations

The MAC operation is the **most frequently executed operation** in DSP algorithms:

```
  accumulator += A × B
```

### Hardware MAC vs Software MAC

```
  Software MAC (no hardware):         Hardware MAC (TriCore):
  ─────────────────────────           ──────────────────────
  LOAD  A                             MADD  result, acc, A, B
  LOAD  B
  MUL   A × B                        → 1 instruction
  ADD   result to accumulator         → Single cycle (pipelined)
  STORE accumulator

  → 5+ instructions
  → Multiple cycles
```

### Where MACs Are Used

| Algorithm | MAC Operations Per Cycle |
|---|---|
| FIR filter (N taps) | N multiply-accumulates per sample |
| FOC motor control | Current vector rotation, PI controller |
| Kalman filter | Matrix multiply (many MACs) |
| Radar CFAR | Cell averaging across range bins |

---

## 13 · Floating Point Unit (FPU)

Select TriCore devices include a hardware **FPU** for IEEE 754 single-precision
floating-point arithmetic.

```
  Without FPU (software emulation):
  ───────────────────────────────────
  float x = a * b;
  → Compiler generates ~10–20 instructions
  → Tens of cycles per operation

  With hardware FPU:
  ──────────────────
  float x = a * b;
  → 1 FPU instruction
  → 1–4 cycles (pipelined)
```

### FPU Applications in Automotive

```
  Vehicle dynamics modeling     →  Position, velocity, acceleration
  Battery state estimation      →  SOC / SOH floating-point models
  Sensor calibration            →  Temperature compensation curves
  ADAS coordinate transforms    →  Camera / radar coordinate systems
```

> ⚠️ Note: Not all TriCore variants include an FPU. Always check the device-specific
> User Manual for FPU availability.

---

## 14 · Pipeline Architecture

TriCore executes instructions through a **multi-stage pipeline** that allows several
instructions to be in-flight simultaneously.

```
                 Cycle:   1    2    3    4    5    6    7
                          │    │    │    │    │    │    │
  Instruction A:        [IF] [ID] [EX] [MA] [WB]
  Instruction B:             [IF] [ID] [EX] [MA] [WB]
  Instruction C:                  [IF] [ID] [EX] [MA] [WB]
  Instruction D:                       [IF] [ID] [EX] [MA] [WB]

  Stages:
  IF  — Instruction Fetch   (read from cache/Flash)
  ID  — Instruction Decode  (identify operation, read registers)
  EX  — Execute             (ALU / DSP operation)
  MA  — Memory Access       (load/store if needed)
  WB  — Write Back          (write result to register)
```

### Pipeline Benefit

Without pipelining, each instruction would require 5 cycles sequentially.
With pipelining, the CPU **completes one instruction per cycle** in steady state.

---

## 15 · Superscalar Execution

Modern TriCore implementations support **superscalar** execution — issuing more than one
instruction per clock cycle.

```
  Scalar (1 instruction/cycle):
  ─────────────────────────────
  Cycle 1:  Instruction A
  Cycle 2:  Instruction B
  Cycle 3:  Instruction C
  Cycle 4:  Instruction D

  Superscalar (2 instructions/cycle, when independent):
  ──────────────────────────────────────────────────────
  Cycle 1:  Instruction A  +  Instruction B  (parallel)
  Cycle 2:  Instruction C  +  Instruction D  (parallel)
```

### Requirement for Superscalar

Instructions must be **independent** — no data dependency between them. The compiler
and hardware scheduler reorder and pair instructions to maximize utilization.

---

## 16 · Branch Prediction

Branches (if/else, loops) disrupt the pipeline because the next instruction to fetch
is not known until the branch is evaluated.

```
  Without branch prediction:
  ──────────────────────────
  Branch instruction in EX stage
        │
        ▼
  Pipeline must stall (or flush wrongly-fetched instructions)
        │
        ▼
  Wasted cycles (pipeline bubbles)

  With branch prediction:
  ───────────────────────
  CPU guesses the branch outcome
        │
        ▼
  Continues fetching predicted path
        │
  ┌─────┴──────┐
  │            │
Correct      Wrong prediction
  │            │
Continue     Flush + refetch correct path
             (penalty cycles)
```

> TriCore includes branch prediction mechanisms to minimize these stalls — critical for
> tight control loops with many conditional checks.

---

## 17 · Memory Architecture

AURIX memory is organized into well-defined regions accessible by the TriCore CPU:

```
  ┌─────────────────────────────────────────────────┐
  │                AURIX Memory Map                 │
  │                                                 │
  │  ┌─────────────────────────────────────────┐    │
  │  │  Program Flash (PFlash)                 │    │
  │  │  Read-only code and constants           │    │
  │  │  ECC protected · Up to several MB       │    │
  │  └─────────────────────────────────────────┘    │
  │                                                 │
  │  ┌─────────────────────────────────────────┐    │
  │  │  Data Flash (DFlash / EEPROM emulation) │    │
  │  │  Non-volatile parameter storage         │    │
  │  │  Calibration data, NVM records          │    │
  │  └─────────────────────────────────────────┘    │
  │                                                 │
  │  ┌─────────────────────────────────────────┐    │
  │  │  SRAM (Local + Global)                  │    │
  │  │  Stack, heap, run-time data             │    │
  │  │  ECC protected                          │    │
  │  └─────────────────────────────────────────┘    │
  │                                                 │
  │  ┌─────────────────────────────────────────┐    │
  │  │  Peripheral Register Space (SFR)        │    │
  │  │  Memory-mapped peripheral control regs  │    │
  │  └─────────────────────────────────────────┘    │
  └─────────────────────────────────────────────────┘
```

---

## 18 · Cache Architecture

AURIX devices include **separate instruction and data caches** per CPU core, reducing
effective memory access latency.

```
  ┌──────────────────────────────────────────────────┐
  │                  CPU Core                        │
  │                                                  │
  │   ┌──────────────────┐  ┌──────────────────┐    │
  │   │  I-Cache         │  │  D-Cache         │    │
  │   │  Instruction     │  │  Data cache      │    │
  │   │  cache           │  │                  │    │
  │   │  (Program Flash) │  │  (SRAM reads)    │    │
  │   └────────┬─────────┘  └────────┬─────────┘    │
  │            │                     │               │
  │            └──────────┬──────────┘               │
  │                       │                          │
  └───────────────────────┼──────────────────────────┘
                          │
                   ┌──────▼──────┐
                   │   Memory    │
                   │  Bus / LMU  │
                   └─────────────┘
```

### Cache Performance Impact

```
  Flash access without cache:  ~5–10 wait states at 300 MHz
  Flash access with I-Cache:   ~0 wait states (cache hit)

  → 5–10× effective throughput improvement on code execution
```

---

## 19 · Memory Protection Unit (MPU)

The MPU enforces **access rights** on memory regions, preventing tasks or cores from
corrupting each other's memory.

```
  ┌────────────────────────────────────────────────┐
  │              MPU Region Configuration          │
  │                                                │
  │  Region 0: Flash (read + execute, no write)    │
  │  Region 1: OS kernel stack (read/write)        │
  │  Region 2: Task A private data (read/write)    │
  │  Region 3: Task B private data (read/write)    │
  │  Region 4: Shared mailbox (read only for Task A│
  │  Region 5: Peripheral SFRs (supervisor only)   │
  │  Region 6: Safety variables (no user access)   │
  └────────────────────────────────────────────────┘
```

### What Happens on a Violation?

```
  Task A attempts write to Task B's private region
                    │
                    ▼
           MPU detects violation
                    │
                    ▼
           Trap generated (Class 4)
                    │
                    ▼
           OS trap handler invoked
                    │
                    ▼
           Faulting task terminated / logged
```

> MPU enforcement is a key requirement for **ASIL-D freedom from interference** between
> software components at different safety levels.

---

## 20 · Privilege Levels

TriCore enforces two privilege levels, controlled by the `PSW.IO` field:

```
  ┌─────────────────────────────────────────────────────┐
  │               TriCore Privilege Levels               │
  │                                                      │
  │  ┌──────────────────────────────────────────┐        │
  │  │  Supervisor Mode (IO = 11b)              │        │
  │  │                                          │        │
  │  │  ✓ Full register access                  │        │
  │  │  ✓ MTCR (write system registers)         │        │
  │  │  ✓ Enable / disable interrupts globally  │        │
  │  │  ✓ Access all peripheral SFRs            │        │
  │  │  → Used by: OS kernel, safety manager    │        │
  │  └──────────────────────────────────────────┘        │
  │                        │                             │
  │              SYSCALL / Trap                          │
  │                        │                             │
  │  ┌──────────────────────────────────────────┐        │
  │  │  User Mode (IO = 00b / 01b)              │        │
  │  │                                          │        │
  │  │  ✗ Cannot modify system registers        │        │
  │  │  ✗ Cannot disable interrupts globally    │        │
  │  │  ✗ Cannot access protected SFRs          │        │
  │  │  → Used by: Application tasks            │        │
  │  └──────────────────────────────────────────┘        │
  └─────────────────────────────────────────────────────┘
```

---

## 21 · Interrupt System Overview

TriCore's interrupt system is built for **low-latency, deterministic response** to
hardware events.

```
  Hardware Event
  (CAN message received, ADC complete, timer expired)
               │
               ▼
       Interrupt Controller (ICU)
       evaluates priority vs current task
               │
         ┌─────┴──────┐
         │            │
     Lower prio    Higher prio
         │            │
     Ignored      CPU suspends current task
    (for now)          │
                       ▼
               Context automatically saved
               to CSA (Context Save Area)
                       │
                       ▼
               ISR (service routine) executes
                       │
                       ▼
               Context restored from CSA
                       │
                       ▼
               Interrupted task resumes
```

> ⚠️ The **Context Save Area (CSA)** mechanism — TriCore's unique approach to
> hardware-managed register saving — will be covered in depth in **Part 3**.

---

## 22 · Trap System Overview

Traps are the TriCore equivalent of hardware exceptions — they fire automatically when
the CPU encounters an illegal or exceptional condition.

### Trap Classes

```
  ┌───────┬─────────────────────────────────────────────────┐
  │ Class │ Trigger Condition                               │
  ├───────┼─────────────────────────────────────────────────┤
  │  1    │ MMU / address translation fault                 │
  │  2    │ Internal protection fault (MPU violation)       │
  │  3    │ Instruction error (undefined opcode)            │
  │  4    │ Context management error (CSA overflow)         │
  │  5    │ Bus error (failed memory access)                │
  │  6    │ Assertion trap (software-triggered)             │
  │  7    │ SYSCALL (system call from user mode)            │
  │  8    │ Non-maskable interrupt (NMI)                    │
  └───────┴─────────────────────────────────────────────────┘
```

### Trap Flow

```
  Exception condition detected
              │
              ▼
  CPU determines trap class and TIN (Trap Identification Number)
              │
              ▼
  Context saved to CSA (same mechanism as interrupt)
              │
              ▼
  Trap vector table entry executed
              │
              ▼
  Safety handler / OS fault handler runs
              │
              ▼
  Decision: recover, reset, or safe state
```

---

## 23 · Multi-Core Processing

TC3xx and TC4xx AURIX devices contain **up to 6 independent TriCore CPUs**, each capable
of running its own program.

```
  ┌──────────────────────────────────────────────────────────┐
  │                    AURIX TC39x                           │
  │                                                          │
  │  ┌────────┐  ┌────────┐  ┌────────┐                      │
  │  │  CPU0  │  │  CPU1  │  │  CPU2  │                      │
  │  │300 MHz │  │300 MHz │  │300 MHz │                      │
  │  │ Safety │  │ Comms  │  │ Diag   │                      │
  │  └────────┘  └────────┘  └────────┘                      │
  │                                                          │
  │  ┌────────┐  ┌────────┐  ┌────────┐                      │
  │  │  CPU3  │  │  CPU4  │  │  CPU5  │                      │
  │  │300 MHz │  │300 MHz │  │300 MHz │                      │
  │  │ App    │  │ App    │  │Monitor │                      │
  │  └────────┘  └────────┘  └────────┘                      │
  │                                                          │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │   LMU — Local Memory Unit (shared global RAM)    │    │
  │  └──────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

### Typical Core Role Assignment

| Core | Typical Role | Example Task |
|---|---|---|
| CPU0 | Safety-critical control | Motor torque, brake pressure |
| CPU1 | Communication | CAN FD RX/TX, Ethernet |
| CPU2 | Diagnostics & monitoring | Self-test, fault logging |
| CPU3–5 | Application / ADAS | Sensor fusion, algorithms |

---

## 24 · Inter-Core Communication

Cores are **isolated** — they cannot directly access each other's local memories. Safe
data exchange uses **shared global memory** via the LMU.

```
  CPU0                      CPU1
  ─────                     ─────
  Write data to LMU   ──►  Read data from LMU

  ┌──────────────┐          ┌──────────────────────────────┐
  │ Shared       │          │ Synchronization Primitives   │
  │ Mailbox in   │  + ───►  │                              │
  │ LMU SRAM     │          │ LDMST  (atomic load+store)   │
  └──────────────┘          │ SWAP   (atomic exchange)     │
                            │ CMPSWAP(compare-and-swap)    │
                            └──────────────────────────────┘
```

### Inter-Core Interrupt (IPI)

```
  CPU0 wants CPU1 to process new data:

  CPU0 → writes to shared flag in LMU
       → triggers SW interrupt on CPU1 via SRC register
  CPU1 → interrupt fires → reads mailbox → processes data
```

---

## 25 · Lockstep Safety

Lockstep is TriCore's primary mechanism for **CPU-level fault detection** at ASIL-D.

```
  ┌──────────────────────────────────────────────────────────┐
  │                   Lockstep Operation                     │
  │                                                          │
  │                   Program Code                           │
  │                        │                                 │
  │          ┌─────────────┴─────────────┐                   │
  │          │                           │                   │
  │   ┌──────▼──────┐             ┌──────▼──────┐            │
  │   │   CPU0      │             │   CPU0 LS   │            │
  │   │  (Master)   │             │  (Checker)  │            │
  │   │             │             │  delayed    │            │
  │   │ Executes    │             │ by N cycles │            │
  │   └──────┬──────┘             └──────┬──────┘            │
  │          │                           │                   │
  │          └─────────────┬─────────────┘                   │
  │                        │                                 │
  │                 ┌──────▼──────┐                          │
  │                 │  Comparator │                          │
  │                 └──────┬──────┘                          │
  │                        │                                 │
  │              ┌──────────┴──────────┐                     │
  │              │                     │                     │
  │           MATCH                MISMATCH                  │
  │              │                     │                     │
  │      Continue normal           SMU Alarm                 │
  │       execution                    │                     │
  │                               Safe Reaction              │
  └──────────────────────────────────────────────────────────┘
```

### What Lockstep Detects

```
  ✓ Bit flips in ALU result registers (radiation, EMI)
  ✓ Stuck-at faults in combinational logic
  ✓ Timing violations causing wrong results
  ✓ Systematic CPU microarchitecture faults
```

### What Lockstep Does NOT Replace

Lockstep protects the **CPU computation**. Separate mechanisms handle:
- Memory errors → ECC
- Software hangs → Watchdog
- Peripheral faults → SMU monitors
- System timing → Clock monitors

---

## 26 · Functional Safety Features

AURIX achieves **ASIL-D** capability through a layered set of hardware safety mechanisms:

```
  ┌──────────────────────────────────────────────────────────┐
  │                 AURIX Safety Layers                      │
  │                                                          │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  CPU Safety          Lockstep execution          │     │
  │  └─────────────────────────────────────────────────┘     │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  Memory Safety       ECC (Flash, RAM, Cache)     │     │
  │  └─────────────────────────────────────────────────┘     │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  Fault Collection    Safety Management Unit      │     │
  │  └─────────────────────────────────────────────────┘     │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  Software Liveness   CPU + Safety Watchdogs      │     │
  │  └─────────────────────────────────────────────────┘     │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  Register Protection ENDINIT lock mechanism      │     │
  │  └─────────────────────────────────────────────────┘     │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  Memory Isolation    MPU (per CPU, configurable) │     │
  │  └─────────────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────┘

  Standard achieved:  ISO 26262  ·  Up to ASIL-D
```

---

## 27 · Security Features

Modern vehicles are internet-connected, making cybersecurity a hardware concern.
AURIX integrates security features directly into silicon:

```
  ┌─────────────────────────────────────────────────────────┐
  │               AURIX Security Features                   │
  │                                                         │
  │  Hardware Security Module (HSM)                         │
  │  ├── Dedicated security CPU core (isolated)             │
  │  ├── AES-128/256 hardware accelerator                   │
  │  ├── RSA / ECC asymmetric crypto                        │
  │  ├── Secure key storage (never readable in plaintext)   │
  │  └── True Random Number Generator (TRNG)                │
  │                                                         │
  │  Secure Boot                                            │
  │  ├── Cryptographic signature check on startup           │
  │  └── Prevents unsigned firmware from running            │
  │                                                         │
  │  Debug Access Protection                                │
  │  ├── JTAG password protection                           │
  │  └── Production-locked devices resist debug access      │
  └─────────────────────────────────────────────────────────┘
```

---

## 28 · Watchdogs

Watchdogs are hardware timers that **must be periodically refreshed** by software.
If software hangs, the watchdog expires and triggers a recovery.

```
  ┌───────────────────────────────────────────────────────┐
  │               AURIX Watchdog System                   │
  │                                                       │
  │  ┌──────────────────────────────────────────────┐     │
  │  │  Safety Watchdog (WDT_S)                     │     │
  │  │  → System-wide · Must be kicked by trusted   │     │
  │  │    safety task · Timeout → system reset       │     │
  │  └──────────────────────────────────────────────┘     │
  │                                                       │
  │  ┌──────────────────────────────────────────────┐     │
  │  │  CPU Watchdog × N  (WDT_CPU0, CPU1 ...)      │     │
  │  │  → One per core · Core-specific timeout      │     │
  │  │  → Detects per-core hangs independently      │     │
  │  └──────────────────────────────────────────────┘     │
  └───────────────────────────────────────────────────────┘
```

### Windowed Watchdog Mode

```
  Standard mode:  kick anytime before timeout  →  OK
  Windowed mode:  must kick ONLY within window  →  too early = FAULT
                                                   too late  = FAULT

  ──────────────────────────────────────────────────►  time
       │          ╔══════════╗          │
       │          ║  Valid   ║          │
       │          ║  Window  ║          │
   Timeout     Open        Close    Timeout
   (too late)  window     window    (too early)
```

> The windowed mode prevents a **runaway loop** from accidentally refreshing the watchdog
> at the wrong time, giving a stronger execution flow guarantee.

---

## 29 · Communication Capabilities

AURIX devices provide one of the most extensive peripheral sets in any automotive MCU:

```
  ┌──────────────────────────────────────────────────────┐
  │          AURIX Communication Peripherals             │
  │                                                      │
  │  ┌─────────────┐  Protocol    Applications           │
  │  │  CAN FD     │  ISO 11898   Body, chassis, engine  │
  │  │  (×4–8 ch)  │              ECU communication      │
  │  └─────────────┘                                     │
  │  ┌─────────────┐                                     │
  │  │  Ethernet   │  100/1000BASE  ADAS, gateway,       │
  │  │             │  -T1 (single  OTA update            │
  │  └─────────────┘  pair)                              │
  │  ┌─────────────┐                                     │
  │  │  FlexRay    │  ISO 17458   Safety networks,       │
  │  │             │              X-by-wire               │
  │  └─────────────┘                                     │
  │  ┌─────────────┐                                     │
  │  │  LIN        │  ISO 17987   Seat, window, HVAC     │
  │  └─────────────┘                                     │
  │  ┌─────────────┐                                     │
  │  │  SPI / QSPI │  —           Sensors, external NVM  │
  │  └─────────────┘                                     │
  │  ┌─────────────┐                                     │
  │  │  I²C        │  —           Simple peripherals     │
  │  └─────────────┘                                     │
  │  ┌─────────────┐                                     │
  │  │  UART/ASCLIN│  —           Debug, diagnostics     │
  │  └─────────────┘                                     │
  └──────────────────────────────────────────────────────┘
```

---

## 30 · Automotive Advantages

Why automotive OEMs and Tier-1 suppliers standardize on TriCore:

```
  ┌──────────────────────┬────────────────────────────────────┐
  │  Advantage           │  Real-World Impact                 │
  ├──────────────────────┼────────────────────────────────────┤
  │  ASIL-D capable      │  Can be used in brake, steering,   │
  │                      │  airbag — no external safety chip  │
  ├──────────────────────┼────────────────────────────────────┤
  │  Deterministic ISA   │  Worst-case timing provable for    │
  │                      │  ISO 26262 timing analysis         │
  ├──────────────────────┼────────────────────────────────────┤
  │  Integrated DSP      │  No external DSP chip needed for   │
  │                      │  motor control or radar            │
  ├──────────────────────┼────────────────────────────────────┤
  │  Multi-core scaling  │  Run safety + comms + app on one   │
  │                      │  MCU, reducing ECU count           │
  ├──────────────────────┼────────────────────────────────────┤
  │  Hardware security   │  Meets UN R155 cybersecurity regs  │
  │                      │  without external crypto chip      │
  ├──────────────────────┼────────────────────────────────────┤
  │  AUTOSAR native      │  Drop into existing automotive     │
  │                      │  software stacks immediately       │
  ├──────────────────────┼────────────────────────────────────┤
  │  AEC-Q100 qualified  │  Guaranteed to survive automotive  │
  │                      │  temperature, vibration, lifetime  │
  └──────────────────────┴────────────────────────────────────┘
```

---

## 31 · TriCore vs ARM Cortex-M

A direct comparison for engineers with ARM background:

| Feature | ARM Cortex-M7 | TriCore (TC3xx) |
|---|---|---|
| **Target market** | General embedded / IoT | Automotive safety systems |
| **ISA type** | ARM Thumb-2 (16/32-bit) | TriCore (16/32-bit mixed) |
| **Register file** | R0–R15 (unified) | D0–D15 + A0–A15 (separate) |
| **DSP support** | Extensions (separate SIMD unit) | Native DSP engine, MADD |
| **FPU** | Optional (Cortex-M4/M7) | Available on select variants |
| **Max cores** | 1 (M7), 2 (M4+M7 on some) | Up to 6 independent CPUs |
| **Lockstep** | Cortex-M33/R5 (optional) | Standard (AURIX-native) |
| **Memory protection** | MPU (optional) | MPU (mandatory in ASIL) |
| **Privilege levels** | Thread / Handler mode | User / Supervisor mode |
| **Interrupt model** | NVIC (nested vectors) | ICU (priority + arbitration) |
| **Context save** | Software pushes to stack | Hardware-managed CSA chain |
| **Functional safety** | Up to ASIL-B typical | ASIL-D full capability |
| **AUTOSAR usage** | Moderate | Extensive (industry standard) |
| **CAN FD** | Some STM32/NXP | Extensive, multi-channel |
| **Hardware security** | TrustZone (optional) | Dedicated HSM core |
| **Toolchain** | GCC, Keil, IAR (well-known) | ADS, TASKING, HighTec, GHS |
| **Learning curve** | Low–Medium | High |

> **The key differentiator:** TriCore's **hardware-managed Context Save Area (CSA)**
> replaces software-managed stack push/pop for interrupt handling — enabling deterministic,
> faster context switches critical for real-time automotive control.

---

## 32 · Summary

TriCore is a purpose-built automotive CPU architecture that can be described by the
convergence of its five key capabilities:

```
         ┌────────────────────────────────────────────┐
         │              TriCore CPU                   │
         │                                            │
         │   COMPUTE          MCU + DSP + RISC        │
         │   ────────         Harvard arch            │
         │                    Superscalar pipeline    │
         │                    FPU, SIMD, MAC          │
         │                                            │
         │   SAFETY           Lockstep execution      │
         │   ──────           ECC on all memory       │
         │                    SMU fault management    │
         │                    MPU isolation           │
         │                                            │
         │   REAL-TIME        Deterministic ISA       │
         │   ─────────        Low-latency IRQ         │
         │                    Hardware context save   │
         │                    Branch prediction       │
         │                                            │
         │   SECURITY         HSM (hardware crypto)   │
         │   ────────         Secure boot             │
         │                    TRNG, key storage       │
         │                                            │
         │   CONNECTIVITY     CAN FD × 8             │
         │   ────────────     Ethernet, FlexRay       │
         │                    SPI, LIN, UART, I²C     │
         └────────────────────────────────────────────┘
```

### What Comes Next in Part 3

The deepest and most unique topic in TriCore internals:

| Topic | Why It Matters |
|---|---|
| **Context Save Areas (CSA)** | TriCore's hardware-managed interrupt/call context — unlike any ARM design |
| **Hardware Context Switching** | How registers are saved/restored without software intervention |
| **Interrupt Entry & Exit** | Cycle-by-cycle detail of what happens when an IRQ fires |
| **Trap Handling** | Hardware exception classification and software reaction |
| **FCX / LCX / PCX / PCXI** | The four registers that manage the CSA linked list |
| **Deterministic IRQ Latency** | How TriCore guarantees worst-case interrupt response |
| **CSA Overflow / Underflow** | What happens when the CSA chain is exhausted |

> These mechanisms are what truly separate TriCore from a fast ARM Cortex-M — and what
> make it capable of meeting hard real-time guarantees in ISO 26262 ASIL-D systems.

---

*Infineon AURIX & TriCore Architecture Series — Part 2 of N*
*Based on publicly available Infineon TriCore Architecture manuals and ISO 26262 principles.*