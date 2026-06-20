# AURIX TriCore Multi-Core Architecture
### Part 6 — Core Structure, Shared Resources, Inter-Core Communication & Lockstep

> A single CPU cannot simultaneously run hard real-time motor control, service CAN/Ethernet
> traffic, run diagnostics, and stay isolated enough for safety certification. AURIX solves
> this by integrating up to 6 independent TriCore CPUs on one die — each with private fast
> memory, all connected through shared resources and a deterministic communication fabric.

---

## Table of Contents

| # | Topic |
|---|---|
| 1 | [Introduction](#1--introduction) |
| 2 | [Why AURIX Uses Multi-Core Architecture](#2--why-aurix-uses-multi-core-architecture) |
| 3 | [TriCore Multi-Core Overview](#3--tricore-multi-core-overview) |
| 4 | [AURIX Core Structure](#4--aurix-core-structure) |
| 5 | [CPU Local Resources](#5--cpu-local-resources) |
| 6 | [Shared System Resources](#6--shared-system-resources) |
| 7 | [System Resource Interconnect (SRI)](#7--system-resource-interconnect-sri) |
| 8 | [Master Core and Slave Cores](#8--master-core-and-slave-cores) |
| 9 | [Multi-Core Startup Sequence](#9--multi-core-startup-sequence) |
| 10 | [Inter-Core Communication Methods](#10--inter-core-communication-methods) |
| 11 | [Shared Memory Communication](#11--shared-memory-communication) |
| 12 | [Synchronization Between Cores](#12--synchronization-between-cores) |
| 13 | [Spinlocks](#13--spinlocks) |
| 14 | [Interrupt-Based Communication](#14--interrupt-based-communication) |
| 15 | [DMA Communication](#15--dma-communication) |
| 16 | [Multi-Core Task Distribution Example](#16--multi-core-task-distribution-example) |
| 17 | [Lockstep Architecture](#17--lockstep-architecture) |
| 18 | [Automotive Example — Vehicle Controller](#18--automotive-example--vehicle-controller) |
| 19 | [Summary](#19--summary) |

---

## 1 · Introduction

Modern vehicles run dozens of concurrent real-time workloads on a single ECU:

```
  ┌──────────────────────────────────────────────────────────┐
  │              Concurrent Automotive Workloads              │
  │                                                            │
  │  ⚡ Motor control (FOC)        — 10–100 µs loop deadline   │
  │  🚗 Engine control             — cylinder-synchronous timing│
  │  👁️  ADAS / radar processing    — high data throughput      │
  │  🌐 Communication gateway      — CAN FD, Ethernet routing  │
  │  🔍 Diagnostics & logging      — background, lower priority│
  │  📡 OTA update management      — large data transfers      │
  └──────────────────────────────────────────────────────────┘
```

### The Single-Core Bottleneck

```
  ┌──────────────────────────┐
  │          CPU0            │
  │                          │
  │  Motor Control  ─┐       │
  │  CAN Handling    ├─ All  │   Problems:
  │  Diagnostics     ├─ on   │   ✗ Limited processing headroom
  │  Communication  ─┘ one   │   ✗ Hard real-time scheduling
  │                   core   │   ✗ High interrupt load on 1 CPU
  └──────────────────────────┘   ✗ No safety isolation between tasks
```

### The AURIX Multi-Core Solution

```
  ┌─────────────────────────────────────────────────────────┐
  │                      AURIX MCU                          │
  │                                                          │
  │  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐      │
  │  │  CPU0  │   │  CPU1  │   │  CPU2  │   │  CPU3  │      │
  │  └────────┘   └────────┘   └────────┘   └────────┘      │
  │                                                          │
  │  Each CPU executes independent software, in parallel,    │
  │  with its own private fast memory                        │
  └─────────────────────────────────────────────────────────┘
```

---

## 2 · Why AURIX Uses Multi-Core Architecture

### Reason 1 — Performance Improvement

```
  Single-core (everything serialized):
  ──────────────────────────────────────
  CPU0
   │
   ├──► Motor Control     ┐
   ├──► CAN                │  All competing for the
   ├──► Diagnostics        │  same CPU time slice
   └──► Communication     ┘

  Multi-core (parallel execution):
  ──────────────────────────────────────
  CPU0 ──► Motor Control      (dedicated, full bandwidth)
  CPU1 ──► Communication      (dedicated, full bandwidth)
  CPU2 ──► Diagnostics        (dedicated, full bandwidth)

  Result: 3× the effective processing capacity for the same clock speed
```

### Reason 2 — Real-Time Determinism

```
  Requirement:  Motor Control ISR must execute every 100 µs — always.

  On a single core sharing with CAN/Ethernet/diagnostics:
  ──────────────────────────────────────────────────────────
  Motor ISR deadline at risk if a long CAN burst or diagnostic
  routine is mid-execution when the 100 µs window arrives.

  On dedicated CPU0 for motor control:
  ──────────────────────────────────────────────────────────
  Motor ISR deadline is isolated from CAN/Ethernet/diagnostic
  load entirely — those run on CPU1/CPU2 and cannot delay it.
```

### Reason 3 — Safety Isolation

```
  ┌───────────────────┐         ┌───────────────────┐
  │       CPU0        │         │       CPU1        │
  │  Safety Control   │         │  Communication    │
  │  (ASIL-D)         │         │  (QM / ASIL-B)    │
  └───────────────────┘         └───────────────────┘

  A bug or crash in CPU1's communication stack
  CANNOT corrupt or stall CPU0's safety control loop.

  This is "freedom from interference" — a core ISO 26262 requirement.
```

---

## 3 · TriCore Multi-Core Overview

```
  ┌────────────────────────────────────────────────────────────────┐
  │                  AURIX TC3xx Multi-Core Layout                 │
  │                                                                │
  │                            SRI Bus                             │
  │                               │                                │
  │     ┌─────────────────────────┼─────────────────────────┐      │
  │     │                         │                         │      │
  │  ┌──▼───┐                 ┌──▼───┐                 ┌──▼───┐   │
  │  │ CPU0 │                 │ CPU1 │                 │ CPU2 │   │
  │  └──┬───┘                 └──┬───┘                 └──┬───┘   │
  │     │                         │                         │      │
  │  ┌──▼───┐                 ┌──▼───┐                 ┌──▼───┐   │
  │  │PSPR0 │                 │PSPR1 │                 │PSPR2 │   │
  │  │DSPR0 │                 │DSPR1 │                 │DSPR2 │   │
  │  │Cache0│                 │Cache1│                 │Cache2│   │
  │  └──────┘                 └──────┘                 └──────┘   │
  │                                                                │
  │                               │                                │
  │                          ┌────▼────┐                           │
  │                          │   LMU   │  ← Shared across all cores│
  │                          └─────────┘                           │
  └────────────────────────────────────────────────────────────────┘
```

Each CPU is a **complete, independent TriCore processor** — not a simplified co-processor.
All cores can run full applications, handle interrupts, and execute the full TriCore ISA.

---

## 4 · AURIX Core Structure

Every TriCore CPU in an AURIX device has an identical internal structure:

```
  ┌──────────────────────────────────────────────────┐
  │                      CPU0                         │
  │                                                   │
  │   ┌─────────────────────────────────────────┐    │
  │   │  Registers  (D0–D15, A0–A15)            │    │
  │   ├─────────────────────────────────────────┤    │
  │   │  CSA Pool   (Context Save Areas)        │    │
  │   ├─────────────────────────────────────────┤    │
  │   │  I-Cache    (Instruction cache)         │    │
  │   ├─────────────────────────────────────────┤    │
  │   │  D-Cache    (Data cache)                │    │
  │   ├─────────────────────────────────────────┤    │
  │   │  PSPR0      (Local program RAM)         │    │
  │   ├─────────────────────────────────────────┤    │
  │   │  DSPR0      (Local data RAM)            │    │
  │   └─────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘

  CPU1, CPU2, CPU3... — identical structure, independent instances
```

> Every CPU is a "complete computer" in miniature — registers, context management,
> cache, and local memory — connected to the rest of the chip only through the SRI bus.

---

## 5 · CPU Local Resources

Resources owned **exclusively** by one CPU — not directly accessible at full speed
by any other core.

### 5.1 · Registers

```
  Fastest storage tier — zero-cycle access within the pipeline

  Used for:
  ✓ Arithmetic and logic operations
  ✓ Address calculation
  ✓ Holding current CPU execution state
```

### 5.2 · CSA (Context Save Area)

```
  Each core owns its own independent CSA pool:

  CPU0 ──► CSA Pool 0   (in DSPR0)
  CPU1 ──► CSA Pool 1   (in DSPR1)
  CPU2 ──► CSA Pool 2   (in DSPR2)

  Used for:
  ✓ Function call context (Upper Context)
  ✓ Interrupt context (Lower Context)
  ✓ Hardware-managed task switching

  See Part 3 for the complete CSA mechanism.
```

### 5.3 · PSPR (Program Scratch-Pad RAM)

```
  CPU0
   │
   ▼
  PSPR0  ──►  BrakeControlISR()   ← copied here for guaranteed timing

  Advantages:
  ✓ ~1–2 cycle instruction fetch (vs 5–10 from Flash)
  ✓ Zero cache-miss possibility
  ✓ Fully deterministic — provable WCET for ISO 26262
```

### 5.4 · DSPR (Data Scratch-Pad RAM)

```
  CPU0
   │
   ▼
  DSPR0  ──►  motorSpeed, taskStacks, CSA pool

  Advantages:
  ✓ ~1–2 cycle data access
  ✓ No SRI bus contention
  ✓ Predictable timing for real-time variables
```

### 5.5 · Local Cache

```
  CPU0 ──► Cache0
  CPU1 ──► Cache1
  CPU2 ──► Cache2

  Accelerates access to:
  • PFlash (via I-Cache)
  • LMU / shared data (via D-Cache)

  ⚠️ Not deterministic like PSPR/DSPR — hit/miss depends on
     prior access history, useful for general code, not WCET-critical paths
```

> Full detail on PSPR, DSPR, Cache, and ECC is covered in **Part 5 — Memory Architecture**.

---

## 6 · Shared System Resources

Resources accessible by **every** CPU core, through the shared bus fabric.

### 6.1 · LMU (Local Memory Unit)

```
  CPU0 ──┐
         │
  CPU1 ──┼──►  LMU  (shared RAM)
         │
  CPU2 ──┘

  Used for:
  ✓ Inter-core communication
  ✓ Shared buffers (e.g., CAN RX queues consumed by multiple cores)
  ✓ Global application/vehicle state
```

### 6.2 · Flash Memory

```
  ┌─────────────────────────────────────────┐
  │  PFlash  ──►  Program code (all cores)  │
  │  DFlash  ──►  Calibration / persistent  │
  │               data (all cores)          │
  └─────────────────────────────────────────┘

  Every CPU can fetch and execute code from the same PFlash image —
  typically each core runs from its own dedicated address range/partition.
```

### 6.3 · Peripherals

```
  Shared peripheral set, reachable by all cores via SPB:

  CAN · CAN FD · SPI · ADC (VADC) · UART (ASCLIN) · Ethernet · LIN · GTM

  Access is arbitrated, and SRN routing (TOS field, see Part 4)
  determines which specific CPU services each peripheral event.
```

---

## 7 · System Resource Interconnect (SRI)

The **SRI** is the high-speed crossbar that connects every CPU, memory, DMA controller,
and peripheral bridge.

```
  ┌───────────────────────────────────────────────────────────┐
  │                       SRI Bus (Crossbar)                  │
  │                                                            │
  │   CPU0  ───┐                                               │
  │   CPU1  ───┤                                               │
  │   CPU2  ───┼──────────►  Memory (PFlash, DFlash, LMU)     │
  │   DMA   ───┤                                               │
  │ Peripheral─┘                                               │
  │                                                            │
  │   Multiple non-conflicting master↔slave pairs              │
  │   can transfer simultaneously (true crossbar behavior)     │
  └───────────────────────────────────────────────────────────┘
```

> Full bus architecture detail (SRI vs SPB, arbitration) is covered in **Part 5**.

---

## 8 · Master Core and Slave Cores

After a system reset, **one core boots first** and is responsible for initializing the
chip before releasing the other cores.

```
  ┌────────────────────────────────────────────────────────┐
  │                 Master / Slave Core Roles               │
  │                                                          │
  │  CPU0  (Master Core)                                    │
  │  ───────────────────                                    │
  │  ✓ Executes first after reset                           │
  │  ✓ Performs hardware initialization                      │
  │  ✓ Configures clocks and PLLs                            │
  │  ✓ Initializes shared memory (LMU, Flash wait states)    │
  │  ✓ Releases CPU1, CPU2, CPU3... to start running         │
  │                                                          │
  │  CPU1 / CPU2 / CPU3...  (Slave Cores)                    │
  │  ─────────────────────────────────                       │
  │  ✓ Held in reset/halt until released by CPU0             │
  │  ✓ Begin executing only after master signals "go"        │
  │  ✓ Run their own independent application code afterward  │
  └────────────────────────────────────────────────────────┘
```

---

## 9 · Multi-Core Startup Sequence

```
  ┌──────────────────────────────────────────────────────┐
  │                       Reset                          │
  └────────────────────────┬───────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────┐
  │                  CPU0 Starts                        │
  │  (CPU1/2/3 held in reset by hardware)                │
  └────────────────────────┬───────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────┐
  │              Initialize System (CPU0)                │
  │  • Clock / PLL setup                                 │
  │  • Watchdog configuration                             │
  │  • Memory wait-state setup                            │
  │  • Safety endinit unlock sequence                     │
  └────────────────────────┬───────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────┐
  │                  Release CPU1                        │
  │  CPU0 writes to CPU1's start register                │
  │  CPU1 begins executing its own startup code           │
  └────────────────────────┬───────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────┐
  │                  Release CPU2                        │
  │  Same mechanism — CPU0 releases CPU2                 │
  └────────────────────────┬───────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────┐
  │       All cores now running independently            │
  └──────────────────────────────────────────────────────┘
```

> This staged startup ensures system-wide resources (clocks, Flash timing, safety
> registers) are configured exactly once, by a single trusted core, before any other
> core begins touching shared hardware.

---

## 10 · Inter-Core Communication Methods

```
  ┌─────────────────────────────────────────────────────────────┐
  │              Inter-Core Communication Toolbox                │
  │                                                                │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │
  │  │ Shared Memory │  │  Interrupts   │  │  Spinlocks    │    │
  │  │   (via LMU)   │  │ (SW-triggered)│  │ (mutual excl.)│    │
  │  └───────────────┘  └───────────────┘  └───────────────┘    │
  │                                                                │
  │  ┌───────────────┐                                            │
  │  │      DMA      │                                            │
  │  │ (CPU-free move│                                            │
  │  │   of data)    │                                            │
  │  └───────────────┘                                            │
  └─────────────────────────────────────────────────────────────┘
```

| Method | Best For | Latency |
|---|---|---|
| Shared memory (LMU) | Passive data sharing (state, config) | Low, but needs sync |
| Software interrupt | Event notification ("data ready") | Very low, immediate |
| Spinlock | Mutual exclusion on a shared resource | Low if uncontended |
| DMA | Bulk data transfer with zero CPU load | Depends on size, async |

---

## 11 · Shared Memory Communication

The simplest inter-core communication pattern: one core writes to LMU, another reads.

```
  CPU0 (Motor Control)                    CPU1 (CAN Gateway)
  ──────────────────────                  ──────────────────────

  Computes vehicle speed
         │
         ▼
  Write to LMU:
  LMU[vehicleSpeed] = 95 km/h
         │
         ▼
  ┌──────────────────┐
  │       LMU         │
  │  vehicleSpeed=95  │
  └──────────────────┘
         │
         ▼
                                          Read from LMU:
                                          speed = LMU[vehicleSpeed]
                                                 │
                                                 ▼
                                          Transmit over CAN FD
```

```
  Flow Summary:
  ──────────────────────────────────────────────────
  CPU0 → Write Data → LMU → CPU1 → Read Data
```

> ⚠️ Remember the **cache coherency hazard** from Part 5 — if CPU1's D-Cache holds a
> stale copy of this address, it may read an old value unless the region is
> non-cacheable or explicitly invalidated.

---

## 12 · Synchronization Between Cores

When multiple cores access the **same shared resource concurrently**, a race condition
can occur.

```
  Unsynchronized access — race condition:

  Shared variable:  counter = 10

  CPU0:                          CPU1:
  ──────                         ──────
  read counter (10)
                                 read counter (10)
  counter = 10 + 1 = 11
                                 counter = 10 + 1 = 11
  write counter = 11
                                 write counter = 11

  Expected result: counter = 12  (two increments)
  Actual result:   counter = 11  ← LOST UPDATE
```

### Solutions

```
  ┌──────────────────────────────────────────────────────────┐
  │  Mechanism            │  How It Prevents the Race          │
  ├──────────────────────────────────────────────────────────┤
  │  Spinlock             │  One core holds exclusive access   │
  │                       │  while others wait (poll)          │
  ├──────────────────────────────────────────────────────────┤
  │  Mutex (OS-level)     │  Like spinlock, but waiting core   │
  │                       │  yields instead of busy-polling    │
  ├──────────────────────────────────────────────────────────┤
  │  Atomic operations    │  LDMST, SWAP, CMPSWAP — hardware   │
  │  (TriCore native)     │  guarantees read-modify-write as   │
  │                       │  one indivisible operation         │
  └──────────────────────────────────────────────────────────┘
```

---

## 13 · Spinlocks

A spinlock grants **exclusive access** to a shared resource — one core proceeds while
others actively wait ("spin") until the lock is released.

```
  ┌──────────────────────────────────────────────────────────┐
  │                    Spinlock Operation                     │
  │                                                            │
  │  CPU0                              CPU1                  │
  │  ─────                             ─────                  │
  │  Acquire Lock                      Try Acquire Lock        │
  │       │                                  │                │
  │       ▼                                  ▼                │
  │  Lock = 1 (success)               Lock already 1           │
  │       │                            → WAIT (spin)           │
  │       ▼                                  │                │
  │  Access Shared Resource                  │  (still spinning)│
  │       │                                  │                │
  │       ▼                                  │                │
  │  Release Lock (Lock = 0)                 │                │
  │       │                                  ▼                │
  │       │                            Lock = 0 → Acquire!     │
  │       │                                  │                │
  │       │                                  ▼                │
  │       │                            Access Shared Resource  │
  └──────────────────────────────────────────────────────────┘
```

### TriCore Atomic Instruction Used for Spinlocks

```c
/* CMPSWAP: atomic compare-and-swap, single indivisible instruction */
uint32_t expected = 0;   /* lock free */
uint32_t desired   = 1;  /* lock taken */

if (atomic_compare_swap(&lock, expected, desired)) {
    /* Lock acquired — critical section */
    sharedCounter++;
    lock = 0;  /* release */
} else {
    /* Lock was already held — spin and retry */
}
```

> Spinlocks are appropriate only for **very short critical sections** — since the
> waiting core burns CPU cycles polling instead of doing useful work.

---

## 14 · Interrupt-Based Communication

One core can directly **notify** another core of an event using a software-triggered
interrupt (covered in detail in Part 4).

```
  CPU0                                CPU1
  ─────                               ─────
  Data ready in shared buffer
        │
        ▼
  Write to GPSR SRN (TOS=CPU1, SETR=1)
        │
        ▼
  Interrupt fires on CPU1
                                       │
                                       ▼
                                 CPU1 ISR executes:
                                 "Process new data"
```

### Common Uses

```
  ✓ "Data ready" notifications (CPU0 finished computing, CPU1 should act)
  ✓ Command dispatch ("CPU2: start diagnostic self-test now")
  ✓ Synchronization barriers between cores at startup
  ✓ Emergency notification ("CPU0: SMU alarm fired, enter safe state")
```

---

## 15 · DMA Communication

For larger data transfers, routing through DMA avoids consuming **any** CPU cycles
on either side.

```
  Without DMA (CPU-mediated transfer):
  ────────────────────────────────────────────────────
  CPU0 reads from its buffer
       │
       ▼
  CPU0 writes to LMU
       │
       ▼
  CPU1 reads from LMU
       │
       ▼
  CPU1 writes to its own buffer

  Cost: CPU0 AND CPU1 cycles consumed


  With DMA:
  ────────────────────────────────────────────────────
  CPU0
   │
   ▼
  Data Buffer (CPU0's DSPR)
   │
   ▼
  DMA Controller  ──────►  CPU1's Memory (DSPR1 or LMU)
   │
   ▼
  (Neither CPU0 nor CPU1 spends cycles on the copy)

  Cost: Zero CPU cycles — DMA does the work in the background
```

### Benefits

```
  ✓ Faster transfer for large data blocks
  ✓ Both source and destination CPUs free to do other work
  ✓ Optional completion interrupt notifies the receiving CPU when done
```

---

## 16 · Multi-Core Task Distribution Example

A realistic 3-core automotive ECU partitioning:

```
  ┌──────────────────────────────────────────────────────────┐
  │                       AURIX ECU                          │
  │                                                            │
  │  CPU0                                                     │
  │   ├──► Motor Control          (hard real-time, ASIL-D)    │
  │   └──► Safety Functions       (lockstep-protected)        │
  │                                                            │
  │  CPU1                                                     │
  │   ├──► CAN FD Communication   (vehicle network)            │
  │   └──► Ethernet               (gateway / diagnostics link) │
  │                                                            │
  │  CPU2                                                     │
  │   ├──► Diagnostics            (UDS services)               │
  │   └──► Logging                (event/fault recording)      │
  └──────────────────────────────────────────────────────────┘
```

Each core's workload is chosen so that:

| Core | Priority Class | Why Isolated |
|---|---|---|
| CPU0 | Hard real-time, safety-critical | Must never be delayed by comms or logging |
| CPU1 | Soft real-time, networking | Bursty traffic shouldn't disturb motor control |
| CPU2 | Best-effort, background | Diagnostics/logging are lowest urgency |

---

## 17 · Lockstep Architecture

Select AURIX cores support **lockstep** — running the same program on two physical
execution units simultaneously and comparing results every cycle, to detect random
hardware faults.

```
  ┌──────────────────────────────────────────────────────────┐
  │                   Lockstep Core Pair                     │
  │                                                            │
  │                       Program Code                        │
  │                            │                               │
  │            ┌───────────────┴───────────────┐               │
  │            │                               │               │
  │      ┌─────▼─────┐                  ┌─────▼─────┐         │
  │      │   CPU0    │                  │  CPU0-LS  │         │
  │      │ (Master)  │                  │ (Checker) │         │
  │      │  Execute  │                  │  Execute  │         │
  │      └─────┬─────┘                  └─────┬─────┘         │
  │            │                               │               │
  │            └───────────────┬───────────────┘               │
  │                            │                               │
  │                     ┌──────▼──────┐                        │
  │                     │ Comparator  │                        │
  │                     └──────┬──────┘                        │
  │                            │                               │
  │                  ┌──────────┴──────────┐                   │
  │                  │                     │                   │
  │               MATCH                MISMATCH                │
  │                  │                     │                   │
  │            Continue                SMU Alarm                │
  │                                  → Safe Reaction             │
  └──────────────────────────────────────────────────────────┘
```

> Lockstep is a **different concept from multi-core task distribution.** Multi-core uses
> separate CPUs running *different* code for *performance*. Lockstep uses a CPU paired
> with a hidden checker core running the *same* code for *safety*. A full deep-dive on
> lockstep and the broader ASIL-D safety architecture is in the dedicated **Safety
> Architecture** document.

---

## 18 · Automotive Example — Vehicle Controller

Putting it all together — a complete multi-core vehicle controller:

```
  ┌────────────────────────────────────────────────────────────────┐
  │                       AURIX Vehicle Controller                 │
  │                                                                  │
  │  ┌────────────┐    ┌────────────┐    ┌────────────┐            │
  │  │   CPU0     │    │   CPU1     │    │   CPU2     │            │
  │  │            │    │            │    │            │            │
  │  │  Motor     │    │  Comms     │    │ Diagnostics│            │
  │  │  Control   │    │  Gateway   │    │            │            │
  │  └─────┬──────┘    └─────┬──────┘    └─────┬──────┘            │
  │        │                  │                  │                  │
  │        └──────────────────┼──────────────────┘                  │
  │                           │                                     │
  │                    ┌──────▼──────┐                              │
  │                    │     LMU     │                              │
  │                    │ Shared      │                              │
  │                    │ Vehicle Data│                              │
  │                    └─────────────┘                              │
  └────────────────────────────────────────────────────────────────┘

  CPU0 writes computed torque/speed → LMU
  CPU1 reads vehicle data → broadcasts over CAN FD to other ECUs
  CPU2 reads vehicle data → logs to DFlash for diagnostic history
```

---

## 19 · Summary

### Private Resources (per CPU)

```
  Registers   D0–D15, A0–A15 — zero-cycle access
  CSA         Independent context pool per core
  Cache       I-Cache + D-Cache, hardware-managed
  PSPR        Local fast program RAM, deterministic
  DSPR        Local fast data RAM, deterministic
```

### Shared Resources (all CPUs)

```
  LMU          Shared RAM for inter-core data
  Flash        PFlash (code) + DFlash (persistent data)
  Peripherals  CAN, SPI, ADC, UART, Ethernet — via SPB
  SRI          High-speed crossbar connecting everything
```

### Communication Methods

```
  Shared Memory   Passive data exchange via LMU
  Interrupts      Active event notification (software-triggered)
  Spinlocks       Mutual exclusion for short critical sections
  DMA             Zero-CPU-cost bulk data transfer
```

### Safety Features

```
  Lockstep          Dual-execution CPU fault detection
  ECC               Memory corruption detection/correction
  Fault Detection   SMU-coordinated alarm and reaction
```

### The Core Equation

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                            │
  │   Multiple Independent Cores                               │
  │              +                                              │
  │   Fast Private Local Memory  (PSPR/DSPR per core)          │
  │              +                                              │
  │   Shared Communication Fabric  (LMU, SRI, SPB)              │
  │              +                                              │
  │   Hardware Safety Mechanisms  (Lockstep, ECC, SMU)          │
  │              =                                              │
  │   Real-Time, Safety-Certified Automotive Computing Platform │
  │                                                              │
  └──────────────────────────────────────────────────────────┘
```

### What Comes Next

| Topic | Why It Matters |
|---|---|
| **GTM (Generic Timer Module)** | PWM generation, motor control timing, input capture |
| **CCU6 / Capture Compare** | 3-phase motor control, encoder interfaces |
| **VADC (Versatile ADC)** | Multi-group ADC architecture and queue handling |
| **AUTOSAR Multi-Core OS** | How OS-level scheduling maps to TriCore's core model |
| **Safety Architecture (ASIL-D)** | Deep dive into lockstep, SMU, and fault reaction |

---

*Infineon AURIX & TriCore Architecture Series — Part 6 of N*
*Based on publicly available Infineon AURIX TC3xx User Manual and TriCore Architecture documentation.*