# AURIX & TriCore Architecture

> **Infineon AURIX** is the dominant automotive microcontroller platform powering
> brake-by-wire, EV drivetrains, ADAS, and more. This guide builds your mental model
> from the ground up — from what TriCore *is*, to how you develop for it.

---

## Table of Contents

| # | Topic |
|---|---|
| 1 | [Introduction](#1--introduction) |
| 2 | [What is Infineon?](#2--what-is-infineon) |
| 3 | [What is AURIX?](#3--what-is-aurix) |
| 4 | [What is TriCore?](#4--what-is-tricore) |
| 5 | [Why Was TriCore Developed?](#5--why-was-tricore-developed) |
| 6 | [TriCore Philosophy](#6--tricore-philosophy) |
| 7 | [AURIX Generations](#7--aurix-generations) |
| 8 | [AURIX Product Families](#8--aurix-product-family-hierarchy) |
| 9 | [Typical Automotive Applications](#9--typical-automotive-applications) |
| 10 | [AURIX Development Boards](#10--aurix-development-boards) |
| 11 | [Development Environment](#11--development-environment) |
| 12 | [Compilers & Toolchains](#12--compilers--toolchains) |
| 13 | [Debuggers](#13--debuggers) |
| 14 | [Software Ecosystem](#14--software-ecosystem) |
| 15 | [AURIX vs STM32](#15--aurix-vs-stm32) |
| 16 | [Learning Roadmap](#16--learning-roadmap) |
| 17 | [Summary](#17--summary) |

---

## 1 · Introduction

The automotive industry has requirements that go far beyond those of typical embedded systems:

```
  General Embedded           Automotive Embedded
  ───────────────            ───────────────────
  ✓ Correct function    →    ✓ Correct function
                        +    ✓ Functional Safety (ISO 26262)
                        +    ✓ Deterministic Real-Time execution
                        +    ✓ Fault detection & safe reaction
                        +    ✓ Multi-core processing
                        +    ✓ Cybersecurity (UN R155)
```

To meet these demands, Infineon developed the **AURIX** microcontroller family, built
on the proprietary **TriCore** CPU architecture.

### Where is AURIX used today?

| Domain | Example System |
|---|---|
| 🚗 Powertrain | Engine Control Units (ECU) |
| ⚡ Electric Vehicles | Motor controller, Battery Management System (BMS) |
| 🛡️ Safety Systems | Brake-by-wire, Electric Power Steering |
| 👁️ ADAS | Radar/LiDAR processing, sensor fusion |
| 🌐 Networking | CAN/Ethernet gateway, OBD diagnostics |
| 🤖 Autonomous Driving | Domain controller, real-time decision-making |

---

## 2 · What is Infineon?

**Infineon Technologies AG** — founded 1999, headquartered in Neubiberg, Germany — is
one of the world's largest automotive semiconductor suppliers.

```
┌─────────────────────────────────────────────────────────────┐
│                    Infineon Product Portfolio               │
│                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐ │
│  │  AURIX MCUs   │  │  XMC Series   │  │  Power Devices  │ │
│  │               │  │               │  │                 │ │
│  │  Automotive   │  │  Industrial   │  │  MOSFETs        │ │
│  │  Safety-grade │  │  Real-time    │  │  IGBTs          │ │
│  │  Multi-core   │  │  Control      │  │  GaN / SiC      │ │
│  └───────────────┘  └───────────────┘  └─────────────────┘ │
│                                                             │
│  ┌───────────────┐  ┌───────────────┐                      │
│  │  Security ICs │  │  Sensors      │                      │
│  │  (SLE/SLx)    │  │  (pressure,   │                      │
│  │  HSM, TPM     │  │   magnetic)   │                      │
│  └───────────────┘  └───────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

**Key markets:** Automotive · Industrial Automation · Power Electronics · IoT · Security

---

## 3 · What is AURIX?

**AURIX** = **AU**tomotive **R**eal-time **I**ntelligence e**X**cellence

It is Infineon's family of high-performance automotive microcontrollers, built from the
ground up to satisfy **ISO 26262** (functional safety) and **AUTOSAR** (software
architecture) requirements.

```
         What makes AURIX different from a generic MCU?
         ──────────────────────────────────────────────

  Generic MCU               AURIX
  ──────────                ──────────────────────────────────────
  Single core          →    Up to 6 TriCore CPUs
  Basic peripherals    →    CAN FD, FlexRay, Ethernet, LIN, SPI...
  No safety HW         →    Lockstep CPU, ECC, SMU, watchdogs
  No security HW       →    Integrated Hardware Security Module (HSM)
  General purpose      →    Automotive qualified, AEC-Q100 grade
```

---

## 4 · What is TriCore?

**TriCore** is the CPU architecture *inside* AURIX. The name reflects its three unified
processor concepts:

```
                    ┌────────────────────┐
                    │      TriCore       │
                    │   CPU Architecture │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
       ┌────────────┐  ┌────────────┐  ┌────────────┐
       │    MCU     │  │    DSP     │  │    RISC    │
       │            │  │            │  │            │
       │ Real-time  │  │  Signal    │  │  General   │
       │ Control    │  │ Processing │  │ Computing  │
       │ I/O, IRQs  │  │ MACs, FFT  │  │ Efficiency │
       └────────────┘  └────────────┘  └────────────┘
```

In a single architecture, TriCore provides:

- **Fast interrupt response** (like a microcontroller)
- **Efficient fixed-point math** (like a DSP)
- **High throughput general computation** (like a RISC processor)

---

## 5 · Why Was TriCore Developed?

### The Problem — Before TriCore

Traditional automotive ECUs required **two separate chips**:

```
  ┌─────────┐         ┌─────────┐
  │   MCU   │◄───────►│   DSP   │
  │         │  Bus    │         │
  │ Control │Interface│  Math   │
  └─────────┘         └─────────┘

  Problems:
  ✗ Higher cost (two chips, two packages)
  ✗ More PCB area required
  ✗ Higher power consumption
  ✗ Latency on inter-chip communication
  ✗ More complex software integration
```

### The Solution — TriCore

```
  ┌──────────────────────────────────┐
  │            TriCore               │
  │                                  │
  │  MCU + DSP + RISC  in ONE core   │
  └──────────────────────────────────┘

  Benefits:
  ✓ Single chip → lower cost
  ✓ Zero inter-chip bus latency
  ✓ Less PCB space
  ✓ Lower power
  ✓ Simpler software model
```

---

## 6 · TriCore Philosophy

Three design principles that govern every TriCore design decision:

```
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │     FAST     │    │     SAFE     │    │ PREDICTABLE  │
  │              │    │              │    │              │
  │ Low-latency  │    │ HW safety    │    │ Deterministic│
  │ interrupts   │    │ mechanisms   │    │ execution    │
  │ DSP-class    │    │ Fault detect │    │ timing       │
  │ math         │    │ & reaction   │    │ Real-time OS │
  │              │    │              │    │ friendly     │
  └──────────────┘    └──────────────┘    └──────────────┘
```

> In automotive systems, **delayed execution = safety hazard**. A brake command that
> arrives 10 ms late due to a software stall is not "a bug" — it is a dangerous failure.
> TriCore's architecture is built to make such delays detectable and preventable.

---

## 7 · AURIX Generations

AURIX has evolved through three major generations, each increasing core count,
performance, and safety capability.

### TC2xx — First Generation

```
  TC21x · TC22x · TC23x · TC26x · TC27x · TC29x
  ────────────────────────────────────────────────
  Cores:       Up to 3 TriCore CPUs
  Safety:      ASIL-D capable
  Comms:       CAN, Ethernet, FlexRay, LIN
  Security:    Hardware Security Module (HSM)
  Status:      Mature / legacy designs
```

### TC3xx — Second Generation ⭐ Most Common Today

```
  TC33x · TC35x · TC36x · TC37x · TC38x · TC39x
  ────────────────────────────────────────────────
  Cores:       Up to 6 TriCore CPUs
  Safety:      Enhanced ASIL-D
  Comms:       CAN FD, Multi-Gigabit Ethernet, FlexRay
  Security:    Enhanced HSM
  Performance: Higher clocks, more Flash & RAM
  Status:      ✓ Active — dominant in current projects
```

### TC4xx — Third Generation (Next-Gen)

```
  TC45x · TC48x · TC49x
  ────────────────────────────────────────────────
  Core Arch:   TriCore 1.8
  Technology:  28 nm process node
  New:         AI/ML acceleration support
  Target:      Software-Defined Vehicle (SDV)
  Status:      Emerging — future-focused designs
```

---

## 8 · AURIX Product Family Hierarchy

Device names follow a systematic naming scheme:

```
  Example device:  TC397XE

  TC  ──►  TriCore-based microcontroller
   3  ──►  Generation (3 = TC3xx)
   9  ──►  Sub-family (performance tier)
   7  ──►  Variant within sub-family
   X  ──►  Package / temperature grade
   E  ──►  Flash / feature option
```

### Family Map

```
  AURIX
    │
    ├── TC2xx  (1st Gen)
    │     ├── TC21x  — Entry level
    │     ├── TC26x  — Mid range
    │     └── TC29x  — High end (3 cores)
    │
    ├── TC3xx  (2nd Gen)  ◄── Most widely used
    │     ├── TC33x  — Entry / cost-optimized
    │     ├── TC37x  — Mid range
    │     └── TC39x  — High end (6 cores)
    │
    └── TC4xx  (3rd Gen)
          ├── TC45x  — Mid range
          └── TC49x  — High end + AI acceleration
```

---

## 9 · Typical Automotive Applications

```
  Application             AURIX Role                        Key Features Used
  ─────────────────────────────────────────────────────────────────────────────
  Engine Control (ECU)    Fuel injection, ignition timing   PWM, ADC, CAN FD
  EV Motor Controller     Torque control, FOC algorithm     Multi-core, PWM, ADC
  Battery Management      Cell monitoring, balancing        ADC, SPI, CAN FD
  Electric Power Steering Torque overlay, ASIL-D control    Lockstep, PWM
  Brake-by-Wire           Pressure control, redundancy      Lockstep, SMU
  ADAS Domain Ctrl        Sensor fusion, object tracking    Multi-core, Ethernet
  CAN/ETH Gateway         Protocol routing, diagnostics     CAN FD, Ethernet, LIN
  Transmission Control    Gear selection, clutch control    PWM, ADC, CAN
```

---

## 10 · AURIX Development Boards

### TC275 Lite Kit — Entry Level

```
  ┌─────────────────────────────────┐
  │       TC275 Lite Kit            │
  │                                 │
  │  MCU:    TC275 (2-core TC2xx)   │
  │  Target: Beginners              │
  │  Use:    Architecture learning  │
  │          Basic peripheral demos │
  └─────────────────────────────────┘
```

### TC375 Lite Kit — Most Popular ⭐

```
  ┌─────────────────────────────────┐
  │       TC375 Lite Kit            │
  │                                 │
  │  MCU:    TC375 (TC3xx family)   │
  │  Cores:  3 TriCore CPUs         │
  │  Comms:  CAN FD, Ethernet       │
  │  Debug:  Onboard MiniWiggler    │
  │  Target: Students, training     │
  │          courses, prototyping   │
  └─────────────────────────────────┘
```

### TC387 Application Kit — Advanced

```
  ┌─────────────────────────────────┐
  │     TC387 Application Kit       │
  │                                 │
  │  MCU:    TC387 (TC3xx family)   │
  │  Target: Advanced projects      │
  │          Automotive prototyping │
  │          Industrial development │
  └─────────────────────────────────┘
```

### TC397 TFT Kit — Full-Featured

```
  ┌─────────────────────────────────┐
  │       TC397 TFT Kit             │
  │                                 │
  │  MCU:    TC397 (6-core TC3xx)   │
  │  Extra:  LCD/TFT display        │
  │  Comms:  CAN FD, Ethernet       │
  │  Target: Multi-core demos       │
  │          Complex system demos   │
  └─────────────────────────────────┘
```

### TC499 Evaluation Board — Next-Gen

```
  ┌─────────────────────────────────┐
  │     TC499 Evaluation Board      │
  │                                 │
  │  MCU:    TC499 (TC4xx family)   │
  │  Target: SDV & AI-enabled       │
  │          next-gen ECU designs   │
  └─────────────────────────────────┘
```

---

## 11 · Development Environment

The standard AURIX development workflow:

```
  ┌─────────────┐
  │  Write Code │  C / C++ with TriCore intrinsics
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Compile   │  TASKING / HighTec GCC / Green Hills / ADS
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │    Flash    │  Via JTAG/DAP using DAS (Device Access Server)
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │    Debug    │  MiniWiggler / Lauterbach TRACE32
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │    Test     │  Unit tests, HIL, functional safety validation
  └─────────────┘
```

---

## 12 · Compilers & Toolchains

### AURIX Development Studio (ADS) — Free / Recommended for Beginners

```
  Provider:   Infineon (official)
  Cost:       Free
  IDE base:   Eclipse
  Includes:   Compiler · Linker · Debugger · Flash programmer
  Best for:   Learning, evaluation, personal projects
  Download:   www.infineon.com/aurix-development-studio
```

### TASKING — Industry Standard

```
  Provider:   Altium / TASKING
  Cost:       Commercial license
  Strengths:  ✓ Highly optimized code generation for TriCore
              ✓ MISRA-C compliance checking
              ✓ AUTOSAR integration
              ✓ Safety qualification documentation
  Best for:   Production automotive ECU development
```

### HighTec GCC — Open Source Based

```
  Provider:   HighTec EDV-Systeme
  Cost:       Free + commercial support options
  Strengths:  ✓ GCC-based (familiar toolchain)
              ✓ Linux / CI/CD pipeline support
              ✓ Eclipse integration
  Best for:   Open-source projects, CI/CD automation
```

### Green Hills MULTI — Safety-Critical

```
  Provider:   Green Hills Software
  Cost:       Commercial license (premium)
  Strengths:  ✓ IEC 61508 / ISO 26262 certified compiler
              ✓ Advanced debugging and profiling
              ✓ Integrity RTOS integration
  Best for:   Certified safety-critical production code
```

---

## 13 · Debuggers

### MiniWiggler — Entry Level

```
  Cost:       Included with Lite Kits (free)
  Interface:  JTAG / DAP
  Capability: Basic flash, run, halt, breakpoints
  Best for:   Learning, simple development
```

### Lauterbach TRACE32 — Industry Standard ⭐

```
  Cost:       Commercial (widely used in OEM/Tier-1)
  Capability: ✓ Multi-core simultaneous debugging
              ✓ Real-time non-intrusive trace (ETM/MTB)
              ✓ OS awareness (FreeRTOS, AUTOSAR OS, etc.)
              ✓ AUTOSAR-aware stack tracing
              ✓ Memory and register visualization
  Best for:   Production ECU debugging, deep system analysis
```

> 💡 Lauterbach TRACE32 is the de-facto standard at automotive OEMs and Tier-1 suppliers
> such as Bosch, Continental, and ZF.

---

## 14 · Software Ecosystem

AURIX software is organized in layers, following the **AUTOSAR** model:

```
  ┌────────────────────────────────────────────┐
  │              Application Layer             │
  │   Motor control, BMS, ADAS logic, etc.     │
  └───────────────────┬────────────────────────┘
                      │
  ┌───────────────────▼────────────────────────┐
  │            AUTOSAR / OS Layer              │
  │   AUTOSAR Classic / Adaptive, FreeRTOS,    │
  │   OSEK, Safety OS                          │
  └───────────────────┬────────────────────────┘
                      │
  ┌───────────────────▼────────────────────────┐
  │           MCAL Driver Layer                │
  │   ADC, PWM, SPI, CAN, ETH, GPT drivers     │
  │   (AUTOSAR-compliant low-level drivers)    │
  └───────────────────┬────────────────────────┘
                      │
  ┌───────────────────▼────────────────────────┐
  │       Infineon-Provided Libraries          │
  │   iLLD (Infineon Low Level Drivers)        │
  │   Safety Libraries · Example Projects      │
  └───────────────────┬────────────────────────┘
                      │
  ┌───────────────────▼────────────────────────┐
  │          AURIX Hardware                    │
  │   TriCore CPUs · Peripherals · Safety HW   │
  └────────────────────────────────────────────┘
```

**iLLD** (Infineon Low Level Drivers) is the key starting point for bare-metal development
— it provides hardware abstraction without requiring a full AUTOSAR stack.

---

## 15 · AURIX vs STM32

A side-by-side comparison for embedded engineers familiar with STM32:

| Feature | STM32 | AURIX |
|---|---|---|
| **CPU Architecture** | ARM Cortex-M (M0 to M7) | Proprietary TriCore (MCU+DSP+RISC) |
| **Primary Market** | General embedded / IoT | Automotive safety systems |
| **Max CPU Cores** | Up to 2 (M4+M7 on some) | Up to 6 TriCore CPUs |
| **Functional Safety** | Limited (some ASIL-B) | Full ASIL-D |
| **Lockstep CPU** | Rare, only select devices | Standard feature |
| **ECC Memory** | Partial, device-dependent | Comprehensive (Flash, RAM, Cache) |
| **Safety Mgmt Unit** | Not present | Dedicated SMU |
| **Watchdog** | Basic IWDG/WWDG | Safety WDT + per-CPU WDT |
| **CAN FD** | Some newer devices | Extensive, multi-channel |
| **Hardware Security** | Some (TrustZone) | Dedicated HSM |
| **AUTOSAR** | Optional, community | Native, industry standard |
| **Development Tools** | STM32CubeIDE (free) | ADS free + TASKING/HighTec (commercial) |
| **Debugging** | ST-Link (simple) | TRACE32, MiniWiggler (complex) |
| **Learning Curve** | Low to Medium | High |
| **Cost (MCU)** | Low–Medium | Medium–High |

> **When to use which:**
> - Use **STM32** for general embedded systems, IoT, industrial, hobby projects.
> - Use **AURIX** when the system must meet automotive safety standards (ISO 26262),
>   requires multi-core real-time processing, or will be deployed in a vehicle.

---

## 16 · Learning Roadmap

A structured path from zero to production-ready AURIX development:

```
  ┌────────────────────────────────────────────────────────┐
  │  Phase 1 — Foundation                                  │
  │                                                        │
  │  Step 1 │ AURIX family structure, generations, naming  │
  │  Step 2 │ TriCore CPU architecture overview            │
  │  Step 3 │ Register architecture (GPR, SFR, CSA)        │
  └────────────────────────────────────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────────┐
  │  Phase 2 — Core Mechanics                              │
  │                                                        │
  │  Step 4 │ Interrupt system (ICU, priority, vectors)    │
  │  Step 5 │ Context Save Area (CSA) — unique to TriCore  │
  │  Step 6 │ Trap system (hardware exception handling)    │
  └────────────────────────────────────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────────┐
  │  Phase 3 — Advanced Features                           │
  │                                                        │
  │  Step 7 │ Multi-core programming (IPI, shared memory)  │
  │  Step 8 │ Communication: CAN FD, LIN, Ethernet         │
  │  Step 9 │ AUTOSAR architecture & MCAL drivers          │
  └────────────────────────────────────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────────┐
  │  Phase 4 — Production Skills                           │
  │                                                        │
  │  Step 10│ Safety concepts: ASIL-D, SMU, lockstep       │
  │  Step 11│ Production ECU development practices         │
  │  Step 12│ ISO 26262 process, functional safety mgmt    │
  └────────────────────────────────────────────────────────┘
```

---

## 17 · Summary

**AURIX** is Infineon's flagship automotive microcontroller platform. Here is the
complete mental model from this guide:

```
  INFINEON
  └── Makes semiconductors for automotive, industrial, power, security
      │
      └── AURIX  (automotive MCU family)
          ├── Name: AUtomotive Real-time Intelligence eXcellence
          ├── Standard: ISO 26262 / ASIL-D
          │
          ├── GENERATIONS
          │   ├── TC2xx  — 1st gen, up to 3 cores, mature
          │   ├── TC3xx  — 2nd gen, up to 6 cores, dominant today  ⭐
          │   └── TC4xx  — 3rd gen, AI-capable, next-gen SDV
          │
          ├── CPU: TriCore Architecture
          │   ├── MCU  — real-time control, I/O, interrupts
          │   ├── DSP  — fixed-point math, signal processing
          │   └── RISC — general computing efficiency
          │
          ├── SAFETY HARDWARE
          │   ├── Lockstep CPU
          │   ├── ECC on all memories
          │   ├── Safety Management Unit (SMU)
          │   └── Hardware Security Module (HSM)
          │
          └── TOOLCHAIN
              ├── ADS        — free IDE (Eclipse-based)
              ├── TASKING    — commercial, industry standard
              ├── HighTec    — GCC-based, open toolchain
              └── Green Hills— certified, safety-critical
```

### What comes next

The natural next step after this overview is the **TriCore CPU internals**:

| Topic | Why It Matters |
|---|---|
| Register Architecture | Foundation of all TriCore assembly and C code |
| Pipeline Design | Understanding execution timing and latency |
| Context Save Areas (CSA) | TriCore's unique interrupt/call stack mechanism |
| Interrupt Handling | Fast, deterministic response to hardware events |
| Trap System | Hardware exception classification and handling |
| Memory Architecture | Segments, MPU, local vs. global memory |
| Multi-Core Operation | Task partitioning, IPC, and synchronization |

---

*Infineon AURIX & TriCore Architecture Series — Part 1 of N*
*Based on publicly available Infineon documentation and ISO 26262 principles.*