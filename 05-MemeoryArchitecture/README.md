# TriCore / AURIX Memory Architecture
### Part 5 — Flash, DSPR, PSPR, LMU, Cache, Buses & ECC

> AURIX does not have a single monolithic RAM block. It has a **distributed memory
> hierarchy** — each CPU owns private local memories for deterministic access, shared
> global RAM for inter-core communication, and a multi-layered bus fabric to connect
> everything. Understanding this architecture is the difference between code that
> works and code that meets real-time deadlines.

---

## Table of Contents

| # | Topic |
|---|---|
| 1 | [Introduction](#1--introduction) |
| 2 | [Why AURIX Memory Is Different](#2--why-aurix-memory-is-different) |
| 3 | [Memory Hierarchy Overview](#3--memory-hierarchy-overview) |
| 4 | [Program Flash (PFlash)](#4--program-flash-pflash) |
| 5 | [Data Flash (DFlash)](#5--data-flash-dflash) |
| 6 | [Program Scratch-Pad RAM (PSPR)](#6--program-scratch-pad-ram-pspr) |
| 7 | [Data Scratch-Pad RAM (DSPR)](#7--data-scratch-pad-ram-dspr) |
| 8 | [Local vs Remote Access](#8--local-vs-remote-access) |
| 9 | [Local Memory Unit (LMU)](#9--local-memory-unit-lmu) |
| 10 | [Cache vs Scratchpad — Why Both Exist](#10--cache-vs-scratchpad--why-both-exist) |
| 11 | [Instruction Cache (I-Cache)](#11--instruction-cache-i-cache) |
| 12 | [Data Cache (D-Cache)](#12--data-cache-d-cache) |
| 13 | [Cache Coherency in Multi-Core Systems](#13--cache-coherency-in-multi-core-systems) |
| 14 | [Memory Performance Comparison](#14--memory-performance-comparison) |
| 15 | [SRI Bus Architecture](#15--sri-bus-architecture) |
| 16 | [SPB Bus Architecture](#16--spb-bus-architecture) |
| 17 | [Memory Arbitration](#17--memory-arbitration) |
| 18 | [ECC (Error Correction Code)](#18--ecc-error-correction-code) |
| 19 | [Memory Placement Strategy](#19--memory-placement-strategy) |
| 20 | [Practical Example — Brake ISR](#20--practical-example--brake-isr) |
| 21 | [Common Interview Questions](#21--common-interview-questions) |
| 22 | [Summary](#22--summary) |

---

## 1 · Introduction

Most embedded engineers are used to a simple memory model:

```
  Typical MCU (e.g. STM32):

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  Flash   │────►│   CPU    │◄───►│   SRAM   │
  └──────────┘     └──────────┘     └──────────┘

  One Flash region, one RAM region, one CPU, one bus.
```

AURIX is architecturally different — and for good reason:

```
  AURIX TC3xx (simplified):

  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  PSPR0   │  │  PSPR1   │  │  PSPR2   │  ← Per-CPU private program RAM
  │  DSPR0   │  │  DSPR1   │  │  DSPR2   │  ← Per-CPU private data RAM
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
  │  CPU0    │  │  CPU1    │  │  CPU2    │
  │ ICache   │  │ ICache   │  │ ICache   │
  │ DCache   │  │ DCache   │  │ DCache   │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
       └──────────────┼──────────────┘
                      │
              ┌───────▼────────┐
              │   SRI Bus      │  ← System Resource Interconnect
              └───────┬────────┘
         ┌────────────┼────────────┐
         │            │            │
    ┌────▼────┐  ┌────▼────┐  ┌───▼──────┐
    │ PFlash  │  │  LMU    │  │   DMA    │
    │ DFlash  │  │(shared) │  │          │
    └─────────┘  └─────────┘  └──────────┘
```

To write correct, real-time, safe automotive software you must understand:

| Memory / Bus | Why It Matters |
|---|---|
| PSPR | Critical ISRs need zero-latency execution — no Flash waits |
| DSPR | Task stacks and local variables with deterministic access |
| LMU | The only RAM all cores share — inter-core communication lives here |
| Cache | Accelerates Flash-based code without code relocation |
| ECC | Makes every memory region safe enough for ASIL-D |
| SRI / SPB | Understand these to reason about bus contention and latency |

---

## 2 · Why AURIX Memory Is Different

### The Problem With a Single Shared RAM

In a multi-core system with one RAM block:

```
  ┌─────────┐          ┌─────────┐          ┌─────────┐
  │  CPU0   │          │  CPU1   │          │  CPU2   │
  └────┬────┘          └────┬────┘          └────┬────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼──────┐
                     │  One RAM    │
                     └─────────────┘

  Problem: All 3 CPUs + DMA compete for the same RAM bus
           → Bus contention → unpredictable latency
           → Real-time deadlines become impossible to guarantee
```

### AURIX's Solution — Private + Shared

```
  Each CPU gets its own private memories (PSPR + DSPR):
  → No bus sharing for local accesses
  → Deterministic access time, guaranteed

  Shared access goes through LMU:
  → All cores can reach it
  → Higher latency than local, but still RAM-speed

  Result: deterministic local operations + flexible shared communication
```

---

## 3 · Memory Hierarchy Overview

Access speed follows physical proximity to the CPU:

```
  ┌────────────────────────────────────────────────────────────┐
  │                   Memory Speed Hierarchy                   │
  │                                                            │
  │  ┌──────────────────────────────────┐                      │
  │  │  CPU Registers                   │  ~1 cycle            │
  │  └──────────────────────────────────┘                      │
  │                    │                                       │
  │  ┌──────────────────────────────────┐                      │
  │  │  PSPR / DSPR  (local scratchpad) │  ~1–2 cycles         │
  │  └──────────────────────────────────┘                      │
  │                    │                                       │
  │  ┌──────────────────────────────────┐                      │
  │  │  I-Cache / D-Cache               │  ~1 cycle (hit)      │
  │  └──────────────────────────────────┘  ~5–10 cycles (miss) │
  │                    │                                       │
  │  ┌──────────────────────────────────┐                      │
  │  │  LMU  (shared global RAM)        │  ~3–6 cycles         │
  │  └──────────────────────────────────┘                      │
  │                    │                                       │
  │  ┌──────────────────────────────────┐                      │
  │  │  PFlash / DFlash                 │  ~5–10+ cycles       │
  │  └──────────────────────────────────┘  (without cache)     │
  │                    │                                       │
  │  ┌──────────────────────────────────┐                      │
  │  │  External Memory (HyperBus etc.) │  >10 cycles          │
  │  └──────────────────────────────────┘                      │
  └────────────────────────────────────────────────────────────┘

  Rule:  Closer to CPU  =  Faster + More Deterministic
```

---

## 4 · Program Flash (PFlash)

**PFlash** is the primary non-volatile storage for program code. Everything the CPU
executes from a cold boot lives here by default.

### What Lives in PFlash

```
  ┌──────────────────────────────────────────────────────────┐
  │                    PFlash Layout                         │
  │                                                          │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │  Startup / Boot Code                             │    │
  │  │  (reset vector, clock init, memory init)         │    │
  │  ├──────────────────────────────────────────────────┤    │
  │  │  Application Code                                │    │
  │  │  (motor control, BMS, ADAS algorithms)           │    │
  │  ├──────────────────────────────────────────────────┤    │
  │  │  MCAL / iLLD Drivers                             │    │
  │  │  (CAN, ADC, PWM, SPI drivers)                    │    │
  │  ├──────────────────────────────────────────────────┤    │
  │  │  AUTOSAR BSW / OS                                │    │
  │  ├──────────────────────────────────────────────────┤    │
  │  │  Lookup Tables / Calibration Data (const)        │    │
  │  └──────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

### PFlash Key Properties

```
  ┌──────────────────────────────────────────────────────────┐
  │  Property            │  Value / Note                     │
  ├──────────────────────────────────────────────────────────┤
  │  Volatility          │  Non-volatile (survives power off) │
  │  Typical size        │  Up to 16 MB (TC39x)              │
  │  Access              │  Read: fast   Write: slow (erase) │
  │  Wait states         │  5–10 cycles without cache        │
  │  ECC                 │  Yes — every read is ECC-checked  │
  │  Execute-in-place    │  Yes — CPU can run code from here  │
  │  Write granularity   │  Page-based (cannot byte-write)   │
  └──────────────────────────────────────────────────────────┘
```

> **The Flash wait-state problem:** At 300 MHz, a 5-cycle Flash access wastes ~17 ns
> on every instruction fetch from Flash — this is why I-Cache and PSPR exist.

---

## 5 · Data Flash (DFlash)

**DFlash** is a separate Flash bank optimized for **persistent data storage** — not code
execution. Think of it as the on-chip EEPROM equivalent.

### What Lives in DFlash

```
  ┌──────────────────────────────────────────────────────────┐
  │                    DFlash Contents                       │
  │                                                          │
  │  ✓ Vehicle configuration parameters                      │
  │  ✓ Calibration tables (engine maps, torque limits)       │
  │  ✓ OBD diagnostic event counters                         │
  │  ✓ Security keys / certificates (if not in HSM)          │
  │  ✓ VIN number and ECU identification                     │
  │  ✓ Odometer / trip counters                              │
  │  ✓ Production end-of-line test results                   │
  └──────────────────────────────────────────────────────────┘
```

### PFlash vs DFlash

```
  ┌────────────────────────────────────────────────────────┐
  │  Property           │  PFlash         │  DFlash        │
  ├────────────────────────────────────────────────────────┤
  │  Primary use        │  Code           │  Data/Config   │
  │  Execute-in-place   │  Yes            │  No            │
  │  Write endurance    │  Lower          │  Higher        │
  │  Erase granularity  │  Larger sectors │  Smaller pages │
  │  Typical use        │  Firmware       │  NVM emulation │
  │  ECC                │  Yes            │  Yes           │
  └────────────────────────────────────────────────────────┘
```

---

## 6 · Program Scratch-Pad RAM (PSPR)

**PSPR** is each CPU's **private, local instruction RAM**. Code copied into PSPR
executes with zero wait states and fully deterministic timing — no cache misses possible.

### Per-Core PSPR Ownership

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   PSPR Ownership Model                      │
  │                                                             │
  │   CPU0 ◄──────► PSPR0   (local, ~1 cycle, CPU0 only fast)  │
  │   CPU1 ◄──────► PSPR1   (local, ~1 cycle, CPU1 only fast)  │
  │   CPU2 ◄──────► PSPR2   (local, ~1 cycle, CPU2 only fast)  │
  │                                                             │
  │   Any CPU can access any PSPR — but cross-core access       │
  │   goes through SRI bus (slower, non-deterministic)          │
  └─────────────────────────────────────────────────────────────┘
```

### What Goes in PSPR

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Code Type                        │  Why PSPR?              │
  ├─────────────────────────────────────────────────────────────┤
  │  Safety-critical ISRs             │  Zero latency, WCET     │
  │  (BrakeISR, SteeringISR)          │  provable for ISO 26262 │
  ├─────────────────────────────────────────────────────────────┤
  │  Motor control algorithms         │  Tight PWM sync loop    │
  │  (FOC, SVPWM inner loop)          │  needs every cycle      │
  ├─────────────────────────────────────────────────────────────┤
  │  Real-time OS scheduler           │  Context switch must be │
  │  tick handler                     │  deterministic          │
  ├─────────────────────────────────────────────────────────────┤
  │  Safety watchdog refresh          │  Must never be delayed  │
  └─────────────────────────────────────────────────────────────┘
```

### PSPR vs Flash Execution

```
  Executing BrakeISR() from PFlash:
  ────────────────────────────────────────────
  Each instruction fetch:
  CPU ──► SRI ──► PFlash ──► (5–10 wait states) ──► CPU
  Worst-case latency: variable (depends on bus load + Flash state)

  Executing BrakeISR() from PSPR0 (on CPU0):
  ────────────────────────────────────────────
  Each instruction fetch:
  CPU ──► PSPR0 ──► (1–2 cycles, no wait states) ──► CPU
  Worst-case latency: fixed, provable, guaranteed
```

---

## 7 · Data Scratch-Pad RAM (DSPR)

**DSPR** is each CPU's **private, local data RAM**. Stack frames, local variables,
RTOS task control blocks, and DMA buffers for a given core all live here.

### Per-Core DSPR Ownership

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   DSPR Ownership Model                      │
  │                                                             │
  │   CPU0 ◄──────► DSPR0   ~up to 240 KB (TC39x)              │
  │   CPU1 ◄──────► DSPR1   ~up to 240 KB                      │
  │   CPU2 ◄──────► DSPR2   ~up to 240 KB                      │
  │                                                             │
  │   DSPR is accessible from other CPUs via SRI —              │
  │   but only the owning CPU gets local (1-cycle) access       │
  └─────────────────────────────────────────────────────────────┘
```

### What Goes in DSPR

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Data Type                      │  Typical Location         │
  ├─────────────────────────────────────────────────────────────┤
  │  Task stacks (OS)               │  DSPR of the owning CPU   │
  │  ISR stacks                     │  DSPR of the target CPU   │
  │  Local variables                │  DSPR (.data / .bss)      │
  │  RTOS TCBs, queues, semaphores  │  DSPR of OS core          │
  │  DMA receive buffers            │  DSPR or LMU              │
  │  CSA pool (Context Save Areas)  │  DSPR (critical!)         │
  └─────────────────────────────────────────────────────────────┘
```

> ⚠️ **The CSA pool must be in DSPR or LMU** — placing it in Flash is illegal.
> Context save/restore is a RAM operation; the CPU writes directly to CSA blocks.

---

## 8 · Local vs Remote Access

This is the **most performance-critical concept** in AURIX memory architecture.

### Local Access — CPU0 → DSPR0

```
  CPU0
   │
   ▼  (direct local path — no bus traversal)
  DSPR0

  Latency:  ~1–2 cycles
  Determinism:  ✓ Guaranteed — no bus arbiter involved
  Bus load:     None
```

### Remote Access — CPU0 → DSPR1

```
  CPU0
   │
   ▼  (must cross the SRI bus)
  SRI Bus
   │
   ▼  (arbiter grants access — variable wait)
  DSPR1

  Latency:  ~5–10+ cycles (depends on bus contention)
  Determinism:  ✗ Variable — depends on what else is on SRI
  Bus load:     Consumes SRI bandwidth
```

### Side-by-Side Impact

```
  ┌──────────────────────────────────────────────────────────┐
  │  Access Type     │  Cycles   │  Deterministic │  Use For │
  ├──────────────────────────────────────────────────────────┤
  │  CPU0 → DSPR0    │  1–2      │  ✓ Yes         │  Stack,  │
  │  (local)         │           │                │  vars,   │
  │                  │           │                │  CSA     │
  ├──────────────────────────────────────────────────────────┤
  │  CPU0 → DSPR1    │  5–10+    │  ✗ Variable    │  Avoid   │
  │  (remote)        │           │                │  for RT  │
  ├──────────────────────────────────────────────────────────┤
  │  CPU0 → LMU      │  3–6      │  ~ Mostly      │  IPC,    │
  │  (shared RAM)    │           │                │  shared  │
  │                  │           │                │  data    │
  ├──────────────────────────────────────────────────────────┤
  │  CPU0 → PFlash   │  5–10+    │  ✗ Variable    │  Avoid   │
  │  (no cache)      │           │  (Flash state) │  for RT  │
  └──────────────────────────────────────────────────────────┘
```

---

## 9 · Local Memory Unit (LMU)

**LMU (Local Memory Unit)** is the **shared global RAM** — accessible by all CPU cores
and the DMA controller. It is the primary medium for safe inter-core communication.

### LMU Connectivity

```
  ┌──────────────────────────────────────────────────────────┐
  │                  LMU Access Model                        │
  │                                                          │
  │        CPU0 ──────┐                                      │
  │        CPU1 ──────┤                                      │
  │        CPU2 ──────┼──► LMU  (up to ~1 MB on TC39x)      │
  │        CPU3 ──────┤                                      │
  │         DMA ──────┘                                      │
  │                                                          │
  │  All masters access LMU through the SRI bus.             │
  │  The SRI arbiter serializes simultaneous requests.       │
  └──────────────────────────────────────────────────────────┘
```

### What Goes in LMU

```
  ┌──────────────────────────────────────────────────────────┐
  │  Data Type                  │  Why LMU?                  │
  ├──────────────────────────────────────────────────────────┤
  │  Inter-core mailboxes       │  Must be reachable by all  │
  │  Shared CAN receive buffers │  CPU1 writes, CPU0 reads   │
  │  Global vehicle state       │  Multiple cores read/write │
  │  AUTOSAR global data        │  Cross-core BSW access     │
  │  DMA transfer buffers       │  DMA writes, CPU reads     │
  └──────────────────────────────────────────────────────────┘
```

### LMU Access Tradeoff

```
  LMU is:
  ✓  Faster than Flash
  ✓  Accessible from all cores and DMA
  ✗  Slower than local DSPR/PSPR
  ✗  Subject to SRI bus contention

  → Use LMU for shared data that must be visible to multiple cores.
  → Do NOT place per-core stacks or CSA pools in LMU.
```

---

## 10 · Cache vs Scratchpad — Why Both Exist

This is a common point of confusion. The short answer:

```
  Cache       =  improve AVERAGE performance automatically
  PSPR/DSPR   =  guarantee WORST-CASE (deterministic) performance
```

### Detailed Comparison

```
  ┌───────────────────────────────────────────────────────────────┐
  │  Property           │  Cache (I$/D$)      │  PSPR / DSPR      │
  ├───────────────────────────────────────────────────────────────┤
  │  Management         │  Hardware automatic │  Software explicit │
  │                     │  (transparent)      │  (linker sections) │
  ├───────────────────────────────────────────────────────────────┤
  │  Hit latency        │  ~1 cycle           │  ~1 cycle         │
  │  Miss latency       │  5–10+ cycles       │  N/A (no misses)  │
  ├───────────────────────────────────────────────────────────────┤
  │  Worst-case timing  │  Unpredictable      │  Fully predictable│
  │                     │  (depends on evict) │                   │
  ├───────────────────────────────────────────────────────────────┤
  │  WCET analysis      │  Complex (cache     │  Trivial          │
  │                     │  state modelling)   │                   │
  ├───────────────────────────────────────────────────────────────┤
  │  ISO 26262          │  Requires analysis  │  Naturally ASIL   │
  │  suitability        │  (WCET harder)      │  friendly         │
  ├───────────────────────────────────────────────────────────────┤
  │  Best for           │  Large bodies of    │  Small, critical  │
  │                     │  general code       │  real-time code   │
  └───────────────────────────────────────────────────────────────┘
```

> **Rule of thumb:** Cache handles the 90% of code that runs infrequently. PSPR/DSPR
> handles the 10% of code where every cycle counts and timing must be provable.

---

## 11 · Instruction Cache (I-Cache)

The **I-Cache** accelerates instruction fetches from PFlash by storing recently-used
code lines in fast on-chip SRAM close to the CPU.

### Without I-Cache

```
  Instruction fetch cycle (no cache):

  CPU requests instruction
          │
          ▼
      SRI Bus
          │
          ▼
      PFlash  (5–10 wait states at 300 MHz)
          │
          ▼
      Instruction delivered
```

### With I-Cache

```
  Cache HIT (subsequent access to same code):

  CPU requests instruction
          │
          ▼
      I-Cache  ──► HIT  ──► ~1 cycle ──► CPU done


  Cache MISS (first access, or evicted line):

  CPU requests instruction
          │
          ▼
      I-Cache  ──► MISS
          │
          ▼
      PFlash  (full wait states — cache line filled)
          │
          ▼
      Cache line stored  ──► CPU gets instruction
          │
          ▼
      Next accesses to same line: all hits (~1 cycle)
```

### When I-Cache Is Most Effective

```
  ✓ Large application code running repeatedly (main control loops)
  ✓ AUTOSAR BSW / OS scheduler called frequently
  ✓ Any code too large to fit in PSPR

  ✗ Code with highly random control flow (many unique branches)
  ✗ Code where WCET must be provable → use PSPR instead
```

---

## 12 · Data Cache (D-Cache)

The **D-Cache** accelerates data reads from LMU and Flash-resident constant data.

```
  CPU reads variable (e.g., from LMU):

  1st access  →  D-Cache MISS  →  LMU access (~5 cycles)  →  line cached
  2nd access  →  D-Cache HIT   →  ~1 cycle  (5× faster)
  3rd access  →  D-Cache HIT   →  ~1 cycle
```

### D-Cache and DMA — The Stale Data Problem

```
  DMA writes new ADC result to LMU address 0xB000_0100
          │
          ▼
  CPU0 reads from 0xB000_0100
          │
          ▼
  D-Cache returns STALE data  ← D-Cache has old value from before DMA write!
          │
          ▼
  CPU0 uses wrong ADC reading  ← SILENT DATA ERROR
```

**Solution:** Mark DMA-target regions as **non-cacheable** in the memory protection
configuration, or explicitly invalidate the D-Cache after a DMA transfer completes.

---

## 13 · Cache Coherency in Multi-Core Systems

Each CPU core has its **own independent cache**. When multiple cores share data through
LMU, cache coherency becomes a critical concern.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                  The Coherency Problem                       │
  │                                                              │
  │  Initial state:  LMU[vehicleSpeed] = 80 km/h                │
  │                                                              │
  │  CPU0 reads vehicleSpeed  ──► Cache0 = 80 km/h              │
  │  CPU1 reads vehicleSpeed  ──► Cache1 = 80 km/h              │
  │                                                              │
  │  CPU0 updates:  LMU[vehicleSpeed] = 95 km/h                 │
  │  CPU0's Cache0 is updated (or written through)              │
  │                                                              │
  │  CPU1 reads vehicleSpeed  ──► Cache1 still = 80 km/h  ✗     │
  │                                  ↑                          │
  │                             STALE DATA — coherency hazard   │
  └──────────────────────────────────────────────────────────────┘
```

### Solutions for Cache Coherency

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Solution                  │  How It Works                   │
  ├──────────────────────────────────────────────────────────────┤
  │  Non-cacheable regions     │  Mark shared LMU regions as     │
  │                            │  non-cacheable in MPU/PMA;      │
  │                            │  every access goes to LMU       │
  ├──────────────────────────────────────────────────────────────┤
  │  Cache flush + invalidate  │  Writer: flush (write back)     │
  │                            │  Reader: invalidate cache line  │
  │                            │  before reading shared data     │
  ├──────────────────────────────────────────────────────────────┤
  │  Memory barriers           │  DSYNC / ISYNC instructions     │
  │  (DSYNC / ISYNC)           │  ensure ordering before/after   │
  │                            │  inter-core communication       │
  ├──────────────────────────────────────────────────────────────┤
  │  Write-through cache mode  │  Writes go to LMU immediately   │
  │                            │  (not just cached)              │
  └──────────────────────────────────────────────────────────────┘
```

> **AUTOSAR rule:** Shared inter-core data (e.g., in a sender-receiver port across cores)
> must use non-cacheable memory or explicit cache management. AUTOSAR OS handles this
> for software components; bare-metal code must manage it manually.

---

## 14 · Memory Performance Comparison

```
  ┌──────────────────────────────────────────────────────────────────┐
  │              Memory Access Latency Reference (TC3xx, 300 MHz)    │
  │                                                                  │
  │  Memory / Path              Approx. Cycles   Deterministic?      │
  │  ─────────────────────────────────────────────────────────────   │
  │  CPU registers              0 (in pipeline)  ✓ Yes               │
  │  PSPR (local, own CPU)      1–2              ✓ Yes               │
  │  DSPR (local, own CPU)      1–2              ✓ Yes               │
  │  I-Cache hit                ~1               ✓ Yes               │
  │  D-Cache hit                ~1               ✓ Yes               │
  │  LMU (no contention)        ~3–6             ~ Mostly            │
  │  LMU (with SRI contention)  up to ~20+       ✗ Variable          │
  │  I-Cache miss (→ PFlash)    ~5–10            ✗ Variable          │
  │  PFlash direct (no cache)   ~5–10            ✗ Variable          │
  │  DSPR remote (via SRI)      ~5–15            ✗ Variable          │
  │  DFlash read                ~10+             ✗ Variable          │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 15 · SRI Bus Architecture

**SRI (System Resource Interconnect)** is the **high-speed system backbone** of AURIX —
the main bus that connects CPUs, memories, and DMA controllers.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                     SRI Bus Topology                             │
  │                                                                  │
  │  Masters (generate requests):       Slaves (serve requests):    │
  │  ──────────────────────────         ─────────────────────────   │
  │  CPU0  ──────┐                      ┌──► PFlash                  │
  │  CPU1  ──────┤                      │                           │
  │  CPU2  ──────┤                      ├──► DFlash                  │
  │  CPU3  ──────┼──► SRI Bus ──────────┤                           │
  │  CPU4  ──────┤   (crossbar)         ├──► LMU                     │
  │  CPU5  ──────┤                      │                           │
  │  DMA   ──────┘                      ├──► DSPR0, DSPR1, DSPR2    │
  │                                     │                           │
  │                                     └──► PSPR0, PSPR1, PSPR2    │
  └──────────────────────────────────────────────────────────────────┘
```

### SRI as a Crossbar Switch

The SRI is not a simple shared bus — it is a **crossbar switch** that can support
multiple simultaneous transfers between different master-slave pairs:

```
  Example — simultaneous transfers on SRI:

  CPU0 → LMU         (reading shared sensor data)
  CPU1 → DSPR2       (remote data access)
  DMA  → PFlash      (instruction prefetch for CPU2)

  If master-slave pairs don't conflict: all proceed in parallel
  If same slave: arbitrated — one waits
```

---

## 16 · SPB Bus Architecture

**SPB (System Peripheral Bus)** is the **lower-speed bus** that connects all standard
peripherals. The CPU reaches peripherals through a bridge from SRI to SPB.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                   SPB Bus Topology                           │
  │                                                              │
  │   CPU                                                        │
  │    │                                                         │
  │    ▼                                                         │
  │   SRI  ──────────────► SRI-to-SPB Bridge                    │
  │                                │                            │
  │                                ▼                            │
  │                          SPB Bus                            │
  │                                │                            │
  │         ┌──────────────────────┼──────────────────────┐     │
  │         │          │           │           │          │     │
  │         ▼          ▼           ▼           ▼          ▼     │
  │        CAN        SPI        UART         ADC        LIN    │
  │       module     module      ASCLIN      VADC       module  │
  │                                                             │
  │   (all standard peripherals hang off SPB)                   │
  └──────────────────────────────────────────────────────────────┘
```

### SRI vs SPB

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Property          │  SRI                  │  SPB            │
  ├──────────────────────────────────────────────────────────────┤
  │  Purpose           │  System backbone      │  Peripheral bus │
  │  Speed             │  High (full clock)    │  Lower (divided)│
  │  Connects          │  CPUs, RAM, Flash,    │  CAN, SPI,      │
  │                    │  DMA, PSPR, DSPR      │  UART, ADC, LIN │
  │  Access latency    │  Low                  │  Higher         │
  │  Typical use       │  Data-critical paths  │  Config & I/O   │
  └──────────────────────────────────────────────────────────────┘
```

---

## 17 · Memory Arbitration

Multiple bus masters (CPUs + DMA) may simultaneously request the same memory slave
(e.g., LMU or PFlash). The **SRI arbiter** resolves conflicts.

```
  Conflict scenario — three masters want LMU at the same moment:

  CPU0 ──────┐
  CPU1 ──────┼──► LMU  ← Only one access per cycle
  DMA  ──────┘

  Arbiter algorithm (simplified, configurable in AURIX):

  Round 1:  DMA  wins  (e.g., highest configured priority)
            CPU0 stalls 1 cycle
            CPU1 stalls 1 cycle

  Round 2:  CPU0 wins  (DMA done, CPU0 wins arbitration)
            CPU1 stalls 1 cycle

  Round 3:  CPU1 wins

  Effect on real-time code:
  ────────────────────────────────────────────────────────────────
  ✗  LMU accesses from real-time ISRs may stall unpredictably
  ✓  Local DSPR/PSPR accesses NEVER stall — no arbiter involved
  → Critical real-time data: always use local DSPR, never LMU
```

---

## 18 · ECC (Error Correction Code)

AURIX applies **ECC to every memory region** — not just Flash. This is a fundamental
requirement for ASIL-D: memory errors must be detected before corrupted data reaches
an actuator command.

### Why Memory Errors Happen

```
  Sources of memory bit errors in automotive:
  ──────────────────────────────────────────────────────────────
  ✗  Cosmic radiation (single-event upsets — SEU)
  ✗  Electromagnetic interference (EMI) from motor inverters
  ✗  Voltage transients during load dumps
  ✗  Temperature extremes accelerating SRAM aging
  ✗  Flash cell charge leakage over time
```

### How ECC Works

```
  WRITE operation:
  ──────────────────────────────────────────────────────────────
  Original data:    1011 0010  (8 bits)
  ECC engine computes parity bits from data
  Stored to memory: 1011 0010 | 10110  (data + ECC bits)


  READ operation — No Error:
  ──────────────────────────────────────────────────────────────
  Read back:        1011 0010 | 10110
  ECC check:        PASS  →  data returned transparently


  READ operation — 1-bit Error (correctable):
  ──────────────────────────────────────────────────────────────
  Stored:           1011 0010
  Bit flip (EMI):   1011 0000   ← bit 1 flipped
  ECC check:        DETECTS single-bit error
                    CORRECTS automatically: 1011 0010
  Result:           Correct data returned  (transparent to SW)
  SMU notification: Optional — can log correctable errors


  READ operation — 2-bit Error (detectable, not correctable):
  ──────────────────────────────────────────────────────────────
  Stored:           1011 0010
  Two bits flipped: 1001 0000
  ECC check:        DETECTS double-bit error
                    CANNOT correct
  Result:           SMU alarm triggered  →  safe reaction
```

### ECC Coverage Across AURIX

```
  ┌──────────────────────────────────────────────────────────────┐
  │              ECC Protection Coverage                         │
  │                                                              │
  │  Memory Region    │  ECC Type          │  ASIL Support       │
  ├──────────────────────────────────────────────────────────────┤
  │  PFlash           │  SECDED (1-bit fix)│  ✓ ASIL-D          │
  │  DFlash           │  SECDED            │  ✓ ASIL-D          │
  │  PSPR             │  SECDED            │  ✓ ASIL-D          │
  │  DSPR             │  SECDED            │  ✓ ASIL-D          │
  │  LMU              │  SECDED            │  ✓ ASIL-D          │
  │  I-Cache          │  SECDED            │  ✓ ASIL-D          │
  │  D-Cache          │  SECDED            │  ✓ ASIL-D          │
  │  SRI bus data     │  Parity            │  ✓ Detection        │
  └──────────────────────────────────────────────────────────────┘

  SECDED = Single Error Correct, Double Error Detect
```

### Why This Matters for Safety

```
  Without ECC:
  ──────────────────────────────────────────────────────────────
  Memory stores:  brakePressure = 100 bar
  Bit flip:       brakePressure = 36 bar   ← wrong, undetected
  Result:         Under-braking, vehicle cannot stop  ← DANGER

  With ECC:
  ──────────────────────────────────────────────────────────────
  1-bit error     → auto-corrected, brakePressure = 100  (safe)
  2-bit error     → SMU alarm → safe state → vehicle stops safely
```

---

## 19 · Memory Placement Strategy

Placing code and data in the right memory is as important as writing correct code.
The linker script (`.lsl` for TASKING, `.ld` for GCC/HighTec) controls placement.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                   Memory Placement Guide                     │
  │                                                              │
  │  MEMORY        PLACE HERE                                    │
  │  ───────────────────────────────────────────────────────     │
  │  PFlash        All general application code, drivers,        │
  │                AUTOSAR BSW, const lookup tables              │
  │                                                              │
  │  DFlash        Non-volatile config, calibration params,      │
  │                OBD counters, VIN, security keys              │
  │                                                              │
  │  PSPR0         CPU0's critical ISRs, motor control inner     │
  │                loop, safety functions, RTOS tick handler     │
  │                                                              │
  │  DSPR0         CPU0's task stacks, local variables, CSA      │
  │                pool for CPU0, ISR-local buffers              │
  │                                                              │
  │  PSPR1 / DSPR1 Same pattern for CPU1's workload             │
  │                                                              │
  │  LMU           Inter-core mailboxes, shared CAN buffers,     │
  │                global vehicle state, AUTOSAR Rte data        │
  └──────────────────────────────────────────────────────────────┘
```

### Linker Section Attribute Example (iLLD / GCC)

```c
/* Place BrakeISR code in CPU0's PSPR */
void __attribute__((section(".cpu0_pspr_code"))) BrakeISR(void)
{
    brakePressure = ADC0_RESULT;
    applyBrake(brakePressure);
}

/* Place brake variable in CPU0's DSPR */
volatile uint32_t __attribute__((section(".cpu0_dspr_data"))) brakePressure;
```

---

## 20 · Practical Example — Brake ISR

A complete placement optimization for a safety-critical brake pressure ISR:

### Before Optimization (All in Flash / Default RAM)

```
  BrakeISR code:      PFlash   → 5–10 cycle fetch latency per instruction
  brakePressure var:  DSPR0    (probably default .bss — acceptable)

  Problem:
  ──────────────────────────────────────────────────────────────
  Each instruction in BrakeISR fetches from Flash:
    - Subject to 5–10 wait states
    - Subject to cache eviction by other code
    - WCET unprovable without complex cache analysis
    - Violates ISO 26262 timing analysis requirements
```

### After Optimization (PSPR + DSPR)

```
  BrakeISR code:      PSPR0    → 1–2 cycle fetch, no wait states ever
  brakePressure var:  DSPR0    → 1–2 cycle data access, local


  Execution path for BrakeISR() after optimization:
  ──────────────────────────────────────────────────────────────

  CAN / ADC interrupt fires
          │
          ▼
  Hardware saves Lower Context → DSPR0 (CSA pool)  ← 1–2 cycles
          │
          ▼
  PC → PSPR0 (BrakeISR entry point)
          │
          ▼
  All instruction fetches: PSPR0  ← 1–2 cycles each
          │
          ▼
  brakePressure read/write: DSPR0  ← 1–2 cycles
          │
          ▼
  RFE — context restored from DSPR0 CSA  ← 1–2 cycles
          │
          ▼
  Interrupted task resumes

  Result:
  ✓ No Flash wait states
  ✓ No cache miss possible
  ✓ Fully deterministic WCET
  ✓ Provable for ISO 26262 timing analysis
```

---

## 21 · Common Interview Questions

### Q: Why does every CPU core have its own DSPR?

```
  Answer:
  Local DSPR gives the owning CPU guaranteed 1–2 cycle data access with
  no SRI bus involvement and no arbitration delay.
  Shared RAM (LMU) introduces variable latency — unacceptable for real-time
  safety functions.
```

### Q: What is the difference between DSPR and LMU?

```
  ┌────────────────────────────────────────────────────────┐
  │  Property          │  DSPR            │  LMU           │
  ├────────────────────────────────────────────────────────┤
  │  Ownership         │  Per-CPU private  │  Shared global │
  │  Access speed      │  ~1–2 cycles      │  ~3–6 cycles   │
  │  Determinism       │  Guaranteed       │  Variable      │
  │  Used for          │  Stack, vars, CSA │  Inter-core IPC│
  │  Other CPUs access │  Via SRI (slow)   │  Via SRI       │
  └────────────────────────────────────────────────────────┘
```

### Q: Why have both cache AND PSPR/DSPR?

```
  Cache:     Automatically improves average-case performance for large
             bodies of code without code relocation. Hit rate ~90%+.
             WCET analysis is complex (cache state must be modelled).

  PSPR/DSPR: Developer explicitly places the most critical code/data
             here. Zero miss possibility. WCET is trivially provable.
             Required for ISO 26262 ASIL-D timing certification.
```

### Q: Why avoid remote DSPR access in real-time code?

```
  Remote DSPR access (CPU0 → DSPR1) must cross the SRI bus.
  SRI bus is shared — DMA, other CPUs, Flash prefetch all compete.
  Arbitration means latency is variable and can spike.
  For hard real-time code (motor control, braking): never use remote DSPR.
```

### Q: What happens if the CSA pool is placed in LMU?

```
  Functionally it may work — but is a design error.
  CSA save/restore (context switch, interrupt handling) becomes
  subject to SRI bus contention.
  Interrupt latency becomes non-deterministic.
  ISO 26262 timing analysis becomes significantly harder.
  → Always place CSA pool in local DSPR.
```

---

## 22 · Summary

### The Complete Memory Map Mental Model

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                 AURIX Memory Architecture Summary                │
  │                                                                  │
  │  PFlash    Non-volatile code storage. Large, ECC-protected.      │
  │            Default home for all application + driver code.       │
  │                                                                  │
  │  DFlash    Non-volatile data storage (NVM emulation). ECC.       │
  │            Config params, calibration, event counters.           │
  │                                                                  │
  │  PSPR      Per-CPU private FAST PROGRAM RAM.                     │
  │            Copy critical ISRs here for deterministic execution.  │
  │                                                                  │
  │  DSPR      Per-CPU private FAST DATA RAM.                        │
  │            Stack, local vars, CSA pool — always local.           │
  │                                                                  │
  │  LMU       Shared global RAM. All cores + DMA can reach it.      │
  │            Use for inter-core mailboxes and shared buffers.      │
  │                                                                  │
  │  I-Cache   Hardware-managed cache for PFlash instructions.       │
  │            Improves average performance; WCET analysis needed.   │
  │                                                                  │
  │  D-Cache   Hardware-managed cache for data (LMU, const Flash).   │
  │            Beware coherency in multi-core + DMA scenarios.       │
  │                                                                  │
  │  SRI Bus   High-speed system backbone (CPUs ↔ RAM ↔ Flash).      │
  │            Crossbar switch; multiple transfers in parallel.      │
  │                                                                  │
  │  SPB Bus   Lower-speed peripheral bus (CAN, SPI, UART, ADC).     │
  │            Bridged from SRI; used for peripheral register access.│
  │                                                                  │
  │  ECC       SECDED protection on ALL memory regions.              │
  │            1-bit: auto-corrected. 2-bit: SMU alarm. ASIL-D.      │
  └──────────────────────────────────────────────────────────────────┘
```

### Decision Tree — Where to Place Code / Data

```
  Is it code or data?
  │
  ├── CODE
  │     Is timing critical / WCET must be proven?
  │     ├── YES  →  PSPR (local to executing CPU)
  │     └── NO   →  PFlash (with I-Cache enabled)
  │
  └── DATA
        Is it non-volatile (survives power cycle)?
        ├── YES  →  DFlash (config, calibration)
        └── NO   (RAM)
              Is it accessed by multiple CPU cores?
              ├── YES  →  LMU (shared, with coherency handling)
              └── NO
                    Is timing critical?
                    ├── YES  →  DSPR (local to owning CPU)
                    └── NO   →  DSPR or LMU (either acceptable)
```

---

*Infineon AURIX & TriCore Architecture Series — Part 5 of N*
*Based on publicly available Infineon AURIX TC3xx User Manual and TriCore Architecture documentation.*