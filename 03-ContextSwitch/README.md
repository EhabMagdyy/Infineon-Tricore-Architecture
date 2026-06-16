# TriCore Context Save Area (CSA) Architecture
### Part 3 — CSA, Upper & Lower Context, Hardware Context Switching & Interrupt Handling

> The **Context Save Area** is the single most unique and misunderstood feature of the
> TriCore architecture. It replaces software stack management with a hardware-linked list
> system — delivering deterministic, ultra-low interrupt latency that stack-based CPUs
> simply cannot match.

---

## Table of Contents

| # | Topic |
|---|---|
| 1 | [Introduction](#1--introduction) |
| 2 | [The Problem CSA Solves](#2--the-problem-csa-solves) |
| 3 | [Traditional Stack-Based Context Saving](#3--traditional-stack-based-context-saving) |
| 4 | [TriCore's CSA Solution](#4--tricores-csa-solution) |
| 5 | [What is a Context?](#5--what-is-a-context) |
| 6 | [What is a CSA?](#6--what-is-a-csa) |
| 7 | [CSA Pool](#7--csa-pool) |
| 8 | [Free Context List (FCX)](#8--free-context-list-fcx) |
| 9 | [Used Context List (PCX)](#9--used-context-list-pcx) |
| 10 | [CSA Control Registers](#10--csa-control-registers) |
| 11 | [Upper Context](#11--upper-context) |
| 12 | [Lower Context](#12--lower-context) |
| 13 | [Why Split Upper and Lower?](#13--why-split-upper-and-lower) |
| 14 | [Function Calls Using CSA](#14--function-calls-using-csa) |
| 15 | [Interrupt Handling Using CSA](#15--interrupt-handling-using-csa) |
| 16 | [Nested Interrupts](#16--nested-interrupts) |
| 17 | [Hardware Context Switching](#17--hardware-context-switching) |
| 18 | [Complete Worked Example](#18--complete-worked-example) |
| 19 | [Context Restoration — Unwinding the Chain](#19--context-restoration--unwinding-the-chain) |
| 20 | [Context Management Traps](#20--context-management-traps) |
| 21 | [Advantages of CSA](#21--advantages-of-csa) |
| 22 | [CSA vs ARM Cortex-M Stack](#22--csa-vs-arm-cortex-m-stack) |
| 23 | [Summary](#23--summary) |

---

## 1 · Introduction

Every processor that supports interrupts and function calls must answer one question:

> _"When execution is interrupted, where do we store the current CPU state so we can
> return to it later?"_

Most processors answer: **the stack**. TriCore answers: **the Context Save Area (CSA)**.

```
  Every Other Architecture          TriCore
  ────────────────────────          ───────────────────────────
  Interrupt fires
        │                                 Interrupt fires
        ▼                                       │
  Software PUSH registers                       ▼
  to stack (takes cycles)           Hardware allocates a CSA block
        │                                       │
        ▼                                       ▼
  ISR executes                      CPU state stored automatically
        │                                       │
        ▼                                       ▼
  Software POP registers            ISR executes
  from stack                                    │
        │                                       ▼
        ▼                           Hardware restores context
  Resume                                        │
                                               ▼
                                          Resume
```

This mechanism is the primary reason AURIX achieves:

| Benefit | What It Enables |
|---|---|
| Extremely low interrupt latency | No software push/pop overhead |
| Deterministic worst-case timing | Hardware operation, not code-path dependent |
| Fast nested interrupt support | Each level gets its own CSA, no stack corruption risk |
| Simplified safety certification | Context save is hardware-verified, not compiler-dependent |

---

## 2 · The Problem CSA Solves

When a **function call** or **interrupt** occurs, the processor must preserve its current
execution state — or it will have no way to resume correctly when control returns.

### What Must Be Saved

```
  Executing: TaskA()
  ─────────────────────────────────────────────────────
  CPU State at the moment of interruption:

  PC    = 0x800_1240   ← Instruction we were about to execute
  PSW   = 0x0000_0980  ← Flags: current privilege, carry, overflow
  A11   = 0x800_1244   ← Return address (where to go back to)
  A10   = 0xD000_0400  ← Stack pointer
  D0–D7               ← Working data registers
  A2–A7               ← Working address/pointer registers
  D8–D15              ← Additional data registers
  A12–A15             ← Additional address registers
```

If **any** of these values are lost or overwritten before the ISR completes,
execution after the ISR will be **undefined** — a safety-critical failure.

---

## 3 · Traditional Stack-Based Context Saving

On ARM Cortex-M, context saving is a combination of hardware and software stack pushes:

```
  Before interrupt — Stack state:
  ┌──────────┐  ← SP (stack pointer)
  │  TaskA   │
  │   data   │
  └──────────┘

  Interrupt fires:
  Hardware automatically pushes 8 registers (xPSR, PC, LR, R12, R3, R2, R1, R0)
  Software ISR prologue pushes remaining callee-saved registers

  Stack during ISR:
  ┌──────────┐  ← SP (moved down)
  │  LR      │  ← pushed by software
  │  R11     │
  │  R10     │
  │  R9      │
  │  R8      │
  │  R7      │  ← pushed by hardware
  │  R6      │
  │  R5      │
  │  R4      │
  │  xPSR    │
  │  PC      │
  │  LR_EXC  │
  │  R12     │
  │  R3      │
  │  R2      │
  │  R1      │
  │  R0      │
  └──────────┘
  │  TaskA   │
  │   data   │
  └──────────┘

  ISR epilogue:
  POP {R8-R11, LR}   ← software restores callee-saved
  Hardware restores remaining 8 on exception return
```

### Problems with Stack-Based Saving

```
  ┌──────────────────────────────────────────────────────────┐
  │  Problem              │  Impact                          │
  ├──────────────────────────────────────────────────────────┤
  │  PUSH/POP take cycles │  Interrupt latency is NOT just   │
  │                       │  the vector fetch — it includes  │
  │                       │  all register save cycles        │
  ├──────────────────────────────────────────────────────────┤
  │  Stack size must be   │  Nested interrupts multiply      │
  │  sized for worst-case │  required stack depth            │
  │  nesting depth        │                                  │
  ├──────────────────────────────────────────────────────────┤
  │  Timing depends on    │  Hard to prove WCET for          │
  │  what the compiler    │  ISO 26262 timing analysis       │
  │  chooses to save      │                                  │
  ├──────────────────────────────────────────────────────────┤
  │  Stack overflow is    │  Silent corruption of task       │
  │  possible             │  data — dangerous in safety      │
  │                       │  systems                         │
  └──────────────────────────────────────────────────────────┘
```

---

## 4 · TriCore's CSA Solution

TriCore replaces software-managed stack saves with a **hardware-managed linked list**
of fixed-size memory blocks called **Context Save Areas**.

```
  TriCore Context Save — The Core Idea
  ──────────────────────────────────────

  Instead of:                      TriCore does:
  ────────────                     ─────────────
  PUSH D0                          Hardware atomically:
  PUSH D1                          1. Takes the head of the free CSA list
  PUSH D2                          2. Writes all context registers into it
  PUSH D3                          3. Links it to the active context chain
  PUSH A11                         4. Updates PCX and FCX
  PUSH A10                         5. Jumps to ISR
  PUSH PSW
  ...
  (N cycles of memory writes)      (Fixed, small number of cycles)
```

The key insight: the number of cycles to enter an interrupt is **constant and
hardware-defined** — not dependent on compiler choices or register usage.

---

## 5 · What is a Context?

A **context** is the complete snapshot of the CPU state needed to resume execution.

```
  ┌──────────────────────────────────────────────────────────┐
  │                    CPU Context = CPU Snapshot            │
  │                                                          │
  │   What was I doing?       →  PC  (Program Counter)      │
  │   What were my flags?     →  PSW (Program Status Word)  │
  │   Where do I return to?   →  A11 (Return Address)       │
  │   Where is my stack?      →  A10 (Stack Pointer)        │
  │   What data was I using?  →  D0–D15 (Data registers)    │
  │   What pointers did I have?→  A0–A15 (Address registers) │
  │   What called me?         →  PCXI (Previous context ptr) │
  └──────────────────────────────────────────────────────────┘
```

Think of context as a **bookmark** — everything you need to reopen a book at the
exact page, paragraph, and word you left at.

---

## 6 · What is a CSA?

A **CSA (Context Save Area)** is a fixed-size, 16-word (64-byte) aligned memory block
that stores exactly one saved context.

```
  One CSA Block (64 bytes, 16-word aligned):

  ┌─────────────────────────────────────────┐  Offset
  │  PCXI  — Previous context link pointer  │   +0
  ├─────────────────────────────────────────┤
  │  PSW   — Program Status Word            │   +4
  ├─────────────────────────────────────────┤
  │  A10   — Stack Pointer                  │   +8
  ├─────────────────────────────────────────┤
  │  A11   — Return Address                 │   +12
  ├─────────────────────────────────────────┤
  │  D8    — Data register                  │   +16
  │  D9                                     │   +20
  │  D10                                    │   +24
  │  D11                                    │   +28
  │  D12                                    │   +32
  │  D13                                    │   +36
  │  D14                                    │   +40
  │  D15                                    │   +44
  ├─────────────────────────────────────────┤
  │  A12   — Address register               │   +48
  │  A13                                    │   +52
  │  A14                                    │   +56
  │  A15                                    │   +60
  └─────────────────────────────────────────┘

  Key properties:
  ✓ Fixed size   — always 64 bytes, no variable-length saves
  ✓ Fixed layout — hardware knows exactly where each register is
  ✓ Hardware managed — CPU fills and reads this, not software
  ✓ Linked — PCXI field points to the previous CSA in the chain
```

---

## 7 · CSA Pool

At startup (usually in the boot code or OS initialization), a region of SRAM is
**reserved and initialized** as a pool of free CSA blocks.

```
  SRAM
  ┌───────────────────────────────────────────────────────┐
  │                    CSA Pool Region                    │
  │                                                       │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
  │  │  CSA  0  │  │  CSA  1  │  │  CSA  2  │   ...      │
  │  │  64 bytes│  │  64 bytes│  │  64 bytes│            │
  │  └──────────┘  └──────────┘  └──────────┘            │
  │                                                       │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
  │  │  CSA  3  │  │  CSA  4  │  │  CSA  5  │   ...      │
  │  │  64 bytes│  │  64 bytes│  │  64 bytes│            │
  │  └──────────┘  └──────────┘  └──────────┘            │
  │                                                       │
  │  All blocks linked into the Free Context List         │
  │  FCX → CSA0 → CSA1 → CSA2 → CSA3 → CSA4 → ... → END │
  └───────────────────────────────────────────────────────┘

  Sizing rule:
  Number of CSAs needed = max call depth + max interrupt nesting depth
  Each function call uses 1 CSA (Upper Context)
  Each interrupt entry uses 1 CSA (Lower Context) + possibly 1 more
```

---

## 8 · Free Context List (FCX)

The **FCX** (Free Context List Head) register always points to the **first available
CSA** in the free pool.

```
  Free list — initial state (all CSAs available):

  FCX
   │
   ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  CSA 0   │────►│  CSA 1   │────►│  CSA 2   │────►│  CSA 3   │
  │ (free)   │     │ (free)   │     │ (free)   │     │ (free)   │
  └──────────┘     └──────────┘     └──────────┘     └──────────┘
                                                           │
                                                          END

  When a context is needed, the CPU:
  1. Takes CSA 0  (head of free list)
  2. Advances FCX to CSA 1  (new head)
  3. Uses CSA 0 to store the context

  After one allocation:

  FCX
   │
   ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  CSA 1   │────►│  CSA 2   │────►│  CSA 3   │
  │ (free)   │     │ (free)   │     │ (free)   │
  └──────────┘     └──────────┘     └──────────┘

  CSA 0 is now in use — allocated to the current context.
```

---

## 9 · Used Context List (PCX)

The **PCX** (Previous Context pointer, stored inside PCXI register) points to the
**chain of active, saved contexts** — one for each call or interrupt nesting level.

```
  Active context chain after 3 nested levels:

  PCX (PCXI)
      │
      ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  CSA 2   │────►│  CSA 1   │────►│  CSA 0   │
  │ ISR ctx  │     │ TaskA ctx│     │ main ctx │
  └──────────┘     └──────────┘     └──────────┘
                                          │
                                         END (NULL)

  Reading the chain tells you the full call/interrupt history:
  Currently executing ISR
  → was interrupted from TaskA
  → which was called from main
```

---

## 10 · CSA Control Registers

Four dedicated TriCore registers manage the entire CSA mechanism:

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   CSA Control Registers                     │
  │                                                             │
  │  ┌────────┬──────────────────────────────────────────────┐  │
  │  │  FCX  │ Free Context List Head                       │  │
  │  │        │ → Points to first available CSA block        │  │
  │  │        │ → Decrements on allocation, increments on    │  │
  │  │        │   deallocation (CSA returned after RFE/RET)  │  │
  │  └────────┴──────────────────────────────────────────────┘  │
  │                                                             │
  │  ┌────────┬──────────────────────────────────────────────┐  │
  │  │  LCX  │ Limit Context Register                       │  │
  │  │        │ → Marks the end of the CSA pool              │  │
  │  │        │ → If FCX reaches LCX → trap (pool exhausted) │  │
  │  └────────┴──────────────────────────────────────────────┘  │
  │                                                             │
  │  ┌────────┬──────────────────────────────────────────────┐  │
  │  │  PCXI │ Previous Context Information Register         │  │
  │  │        │ → Contains pointer to previous saved CSA     │  │
  │  │        │ → Also carries: UL bit (Upper/Lower flag),   │  │
  │  │        │   PIE (previous interrupt enable), PCPN      │  │
  │  │        │   (previous CPU priority number)             │  │
  │  └────────┴──────────────────────────────────────────────┘  │
  │                                                             │
  │  ┌────────┬──────────────────────────────────────────────┐  │
  │  │  PC   │ Program Counter                              │  │
  │  │        │ → Saved into/restored from CSA during        │  │
  │  │        │   context switch (not directly CSA register  │  │
  │  │        │   but part of the saved state)               │  │
  │  └────────┴──────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────┘
```

### PCXI Register Bit Layout

```
  PCXI  (32-bit register):

  Bits [31:22]  PCPN  — Previous CPU Priority Number
  Bit  [21]     PIE   — Previous Interrupt Enable state
  Bit  [20]     UL    — Upper/Lower context type flag (0=lower, 1=upper)
  Bits [19:0]   PCX   — Previous CSA segment:offset pointer
```

---

## 11 · Upper Context

TriCore divides the saved context into **two halves** — Upper and Lower. The
**Upper Context** contains the registers that are preserved across function calls
(callee-saved in the TriCore ABI).

```
  Upper Context CSA Layout:

  ┌─────────────────────────────────────────────────────┐
  │  PCXI  — Link to previous context in chain          │
  ├─────────────────────────────────────────────────────┤
  │  PSW   — Program Status Word (flags, privilege)     │
  ├─────────────────────────────────────────────────────┤
  │  A10   — Stack Pointer (SP)                         │
  ├─────────────────────────────────────────────────────┤
  │  A11   — Return Address (RA)                        │
  ├─────────────────────────────────────────────────────┤
  │  D8    ┐                                            │
  │  D9    │                                            │
  │  D10   │  Data registers D8–D15                     │
  │  D11   │  (callee-saved data)                       │
  │  D12   │                                            │
  │  D13   │                                            │
  │  D14   │                                            │
  │  D15   ┘                                            │
  ├─────────────────────────────────────────────────────┤
  │  A12   ┐                                            │
  │  A13   │  Address registers A12–A15                 │
  │  A14   │  (callee-saved pointers)                   │
  │  A15   ┘                                            │
  └─────────────────────────────────────────────────────┘

  UL bit in PCXI = 1  →  this CSA holds an Upper Context
  Saved during: CALL instruction (function calls)
```

---

## 12 · Lower Context

The **Lower Context** contains the caller-saved (scratch) registers — those that
must be preserved across an interrupt boundary but are not preserved across a
normal function call.

```
  Lower Context CSA Layout:

  ┌─────────────────────────────────────────────────────┐
  │  PCXI  — Link to previous context in chain          │
  ├─────────────────────────────────────────────────────┤
  │  A11   — Return Address (saved interrupt return PC) │
  ├─────────────────────────────────────────────────────┤
  │  D0    ┐                                            │
  │  D1    │                                            │
  │  D2    │  Data registers D0–D7                      │
  │  D3    │  (caller-saved scratch registers)          │
  │  D4    │                                            │
  │  D5    │                                            │
  │  D6    │                                            │
  │  D7    ┘                                            │
  ├─────────────────────────────────────────────────────┤
  │  A2    ┐                                            │
  │  A3    │  Address registers A2–A7                   │
  │  A4    │  (caller-saved pointer registers)          │
  │  A5    │                                            │
  │  A6    │                                            │
  │  A7    ┘                                            │
  └─────────────────────────────────────────────────────┘

  UL bit in PCXI = 0  →  this CSA holds a Lower Context
  Saved during: Interrupt entry (BISR / hardware interrupt mechanism)
```

---

## 13 · Why Split Upper and Lower?

```
  ┌─────────────────────────────────────────────────────────────┐
  │             Upper vs Lower — Design Rationale               │
  │                                                             │
  │  Scenario 1: Normal function call (CALL instruction)        │
  │  ────────────────────────────────────────────────────────   │
  │  Callee must preserve: D8–D15, A10–A15, PSW                │
  │  Callee is free to use: D0–D7, A2–A7 (scratch)             │
  │                                                             │
  │  → Save Upper Context only (1 CSA)                          │
  │  → No need to save scratch registers D0–D7, A2–A7          │
  │  → Caller will restore those itself if needed               │
  │                                                             │
  │  Scenario 2: Interrupt entry                                │
  │  ────────────────────────────────────────────────────────   │
  │  ISR can use any register freely                            │
  │  Interrupted code's scratch registers WILL be clobbered    │
  │                                                             │
  │  → Save Lower Context first (D0–D7, A2–A7) = 1 CSA         │
  │  → Upper Context already saved by the call chain            │
  │                                                             │
  │  Result: Most transitions need only 1 CSA allocation        │
  │  instead of saving all 32 registers every time             │
  │                                                             │
  │  Performance gain: ~50% reduction in context save memory    │
  │  traffic for the common case (function calls)              │
  └─────────────────────────────────────────────────────────────┘
```

---

## 14 · Function Calls Using CSA

Tracing a `CALL` instruction through the CSA mechanism:

```
  Code:
  ─────────────────
  void main() {
      TaskA();      ← CALL instruction here
  }

  Before CALL:
  ──────────────────────────────────────────────────────
  FCX ──► CSA0 ──► CSA1 ──► CSA2 ──► CSA3
  PCXI = NULL   (no previous context)

  CALL TaskA executes:
  ──────────────────────────────────────────────────────
  1. CPU takes CSA0 from free list  (FCX advances to CSA1)
  2. Saves Upper Context into CSA0:
       CSA0.PCXI = NULL    ← previous context pointer
       CSA0.PSW  = current PSW
       CSA0.A10  = current SP
       CSA0.A11  = return address (instruction after CALL)
       CSA0.D8–D15, A12–A15
  3. Sets PCXI to point at CSA0
  4. Jumps to TaskA entry point

  After CALL (inside TaskA):
  ──────────────────────────────────────────────────────
  FCX ──► CSA1 ──► CSA2 ──► CSA3     (CSA0 now used)
  PCXI ──► CSA0 [main() upper context]

  Execution stack:
  main()
    └──► TaskA()   (currently running)
```

---

## 15 · Interrupt Handling Using CSA

When an interrupt fires, the CPU saves a **Lower Context** (scratch registers) using the
CSA mechanism — not a software stack push:

```
  Scenario: CAN RX interrupt fires while TaskB() is running

  Before interrupt:
  ──────────────────────────────────────────────────────
  FCX  ──► CSA2 ──► CSA3
  PCXI ──► CSA1 [TaskA upper ctx] ──► CSA0 [main upper ctx]
  Currently executing: TaskB()

  Interrupt entry sequence (hardware, ~5 cycles):
  ──────────────────────────────────────────────────────
  Step 1:  CPU takes CSA2 from free list  (FCX → CSA3)
  Step 2:  Saves Lower Context into CSA2:
             CSA2.PCXI = old PCXI value   ← links to TaskA chain
             CSA2.A11  = interrupted PC   ← where to return
             CSA2.D0–D7, A2–A7
             UL bit = 0  (lower context)
  Step 3:  PCXI now points to CSA2
  Step 4:  CPU loads ICR (interrupt priority), sets new priority
  Step 5:  Jumps to CAN_ISR vector

  After interrupt entry:
  ──────────────────────────────────────────────────────
  FCX  ──► CSA3
  PCXI ──► CSA2 [TaskB lower ctx]
              └──► CSA1 [TaskA upper ctx]
                     └──► CSA0 [main upper ctx]

  Execution call tree:
  main()
    └──► TaskA()
           └──► TaskB()    ← interrupted here
                  └──► CAN_ISR()   (now running)
```

---

## 16 · Nested Interrupts

A higher-priority interrupt can preempt a running ISR. Each level gets its own CSA.

```
  Scenario: BRAKE_ISR (higher priority) fires during CAN_ISR

  State before BRAKE interrupt:
  ──────────────────────────────────────────────────────
  FCX  ──► CSA3
  PCXI ──► CSA2 [TaskB lower] ──► CSA1 [TaskA upper] ──► CSA0 [main upper]
  Running: CAN_ISR()

  BRAKE_ISR entry (hardware allocates CSA3):
  ──────────────────────────────────────────────────────
  FCX  = NULL   (pool exhausted in this 4-CSA example)
  PCXI ──► CSA3 [CAN_ISR lower]
              └──► CSA2 [TaskB lower]
                     └──► CSA1 [TaskA upper]
                            └──► CSA0 [main upper]

  Full nesting visualization:
  ──────────────────────────────────────────────────────
        PCX (PCXI)
            │
            ▼
         ┌──────┐
         │ CSA3 │  ← BRAKE_ISR context (currently running)
         └──┬───┘
            │
            ▼
         ┌──────┐
         │ CSA2 │  ← CAN_ISR lower context
         └──┬───┘
            │
            ▼
         ┌──────┐
         │ CSA1 │  ← TaskA upper context
         └──┬───┘
            │
            ▼
         ┌──────┐
         │ CSA0 │  ← main() upper context
         └──────┘
            │
           NULL
```

---

## 17 · Hardware Context Switching

The complete hardware sequence that occurs on **every context switch** (call or interrupt):

```
  ┌──────────────────────────────────────────────────────────────┐
  │              Hardware Context Save Sequence                  │
  │                 (~5 cycles, fixed, deterministic)            │
  │                                                              │
  │  1. Determine context type                                   │
  │     CALL       → Upper Context (UL=1)                        │
  │     Interrupt  → Lower Context (UL=0)                        │
  │                                                              │
  │  2. Allocate CSA from free list                              │
  │     new_csa = memory[FCX]                                    │
  │     FCX = new_csa.next_free    ← advance free list head      │
  │                                                              │
  │  3. Write context registers into CSA                         │
  │     new_csa.PCXI = PCXI        ← link to previous context   │
  │     new_csa.PSW  = PSW                                       │
  │     new_csa.A10  = A10  (SP)                                 │
  │     new_csa.A11  = A11  (RA / interrupted PC)               │
  │     new_csa.Dx   = Dx   (relevant data registers)           │
  │     new_csa.Ax   = Ax   (relevant address registers)        │
  │                                                              │
  │  4. Update PCXI to point to new CSA                          │
  │     PCXI = new_csa address + UL bit + PIE + PCPN            │
  │                                                              │
  │  5. Load new execution context                               │
  │     PC = target address (function or ISR vector)             │
  │     For interrupt: update ICR with new priority              │
  │                                                              │
  │  Software does NONE of the above — it is entirely hardware   │
  └──────────────────────────────────────────────────────────────┘
```

---

## 18 · Complete Worked Example

A full trace from initial state through 2 function calls + 2 nested interrupts.

### Initial State

```
  Free list:
  FCX ──► CSA0 ──► CSA1 ──► CSA2 ──► CSA3 ──► END
  PCXI = NULL
  Running: main()
```

### Step 1 — `main()` calls `TaskA()`

```
  CALL instruction: allocates CSA0 (Upper Context)

  FCX  ──► CSA1 ──► CSA2 ──► CSA3
  PCXI ──► CSA0 [main UC]
  Running: TaskA()
```

### Step 2 — `TaskA()` calls `TaskB()`

```
  CALL instruction: allocates CSA1 (Upper Context)

  FCX  ──► CSA2 ──► CSA3
  PCXI ──► CSA1 [TaskA UC] ──► CSA0 [main UC]
  Running: TaskB()
```

### Step 3 — CAN RX Interrupt fires

```
  Hardware interrupt entry: allocates CSA2 (Lower Context)

  FCX  ──► CSA3
  PCXI ──► CSA2 [TaskB LC] ──► CSA1 [TaskA UC] ──► CSA0 [main UC]
  Running: CAN_ISR()
```

### Step 4 — BRAKE Interrupt fires (higher priority)

```
  Hardware interrupt entry: allocates CSA3 (Lower Context)

  FCX  = NULL  (pool exhausted)
  PCXI ──► CSA3 [CAN_ISR LC] ──► CSA2 [TaskB LC] ──► CSA1 [TaskA UC] ──► CSA0 [main UC]
  Running: BRAKE_ISR()
```

### Peak State — All 4 CSAs In Use

```
  Active context chain (read top to bottom = innermost to outermost):

  PCXI
   │
   ▼
  ┌────────────────────────────────────────┐
  │  CSA3 — BRAKE_ISR Lower Context        │  ← Currently executing
  │  (CAN_ISR interrupted PC, D0–D7, A2–7) │
  └──────────────────┬─────────────────────┘
                     │
   ▼
  ┌────────────────────────────────────────┐
  │  CSA2 — TaskB / CAN_ISR Lower Context  │  ← CAN_ISR was here
  │  (TaskB interrupted PC, D0–D7, A2–7)   │
  └──────────────────┬─────────────────────┘
                     │
   ▼
  ┌────────────────────────────────────────┐
  │  CSA1 — TaskA Upper Context            │  ← TaskA was running
  │  (PSW, SP, RA, D8–15, A10–15)          │
  └──────────────────┬─────────────────────┘
                     │
   ▼
  ┌────────────────────────────────────────┐
  │  CSA0 — main() Upper Context           │  ← Bottom of the chain
  │  (PSW, SP, RA, D8–15, A10–15)          │
  └──────────────────┬─────────────────────┘
                     │
                    NULL
```

---

## 19 · Context Restoration — Unwinding the Chain

Each `RFE` (Return from Exception) or `RET` instruction unwinds one level:

### BRAKE_ISR returns (RFE)

```
  Before:
  PCXI ──► CSA3 ──► CSA2 ──► CSA1 ──► CSA0

  RFE executes:
  1. Read context from CSA3
  2. Restore D0–D7, A2–A7, A11 (return PC), PSW
  3. Return CSA3 to free list: FCX ← CSA3
  4. Set PCXI = CSA3.PCXI  (now points to CSA2)
  5. Jump to restored PC

  After:
  FCX  ──► CSA3 (returned to free list)
  PCXI ──► CSA2 ──► CSA1 ──► CSA0
  Running: CAN_ISR()  (resumed at interrupted point)
```

### CAN_ISR returns (RFE)

```
  FCX  ──► CSA2 ──► CSA3
  PCXI ──► CSA1 ──► CSA0
  Running: TaskB()  (resumed at interrupted instruction)
```

### TaskB() returns (RET)

```
  FCX  ──► CSA1 ──► CSA2 ──► CSA3
  PCXI ──► CSA0
  Running: TaskA()  (resumed at instruction after CALL TaskB)
```

### TaskA() returns (RET)

```
  FCX  ──► CSA0 ──► CSA1 ──► CSA2 ──► CSA3   (fully restored!)
  PCXI = NULL
  Running: main()  (resumed at instruction after CALL TaskA)
```

> **The free list is exactly back to its original state.** CSAs are borrowed and
> returned like tickets — no memory leak, no fragmentation, no corruption possible.

---

## 20 · Context Management Traps

The CSA pool is **finite**. If the pool is exhausted, the CPU triggers a hardware trap.

### Trap: FCX Reaches LCX (Pool Nearly Empty)

```
  FCX ──► (last CSA before LCX boundary)
               │
               ▼
       CPU generates Trap Class 4
       TIN 1 — Context List Depletion Warning

  This is a warning: 1 CSA remaining.
  Software must react (reduce nesting, reset).
```

### Trap: Context Management Error

```
  FCX = NULL and another CALL/interrupt occurs:
               │
               ▼
       CPU generates Trap Class 4
       TIN 3 — Free Context List Exhaustion

  System cannot continue safely.
  OS trap handler must:
  → Log the fault
  → Trigger system reset
  → Enter safe state
```

### Root Causes

```
  ┌────────────────────────────────────────────────────────┐
  │  Cause                    │  Fix                       │
  ├────────────────────────────────────────────────────────┤
  │  Infinite recursion       │  Fix algorithm logic       │
  │  Too many nested IRQs     │  Reduce nesting depth      │
  │  CSA pool too small       │  Increase pool at startup  │
  │  ISR not returning        │  Fix ISR completion bug    │
  └────────────────────────────────────────────────────────┘
```

---

## 21 · Advantages of CSA

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    CSA Advantage Summary                     │
  │                                                              │
  │  ✓ Hardware-assisted save/restore                            │
  │    No software PUSH/POP instructions — faster, simpler       │
  │                                                              │
  │  ✓ Deterministic interrupt latency                           │
  │    Fixed number of cycles regardless of call depth           │
  │    or number of registers in use — provable WCET             │
  │                                                              │
  │  ✓ No stack overflow corruption                              │
  │    CSA pool depletion triggers a trap before any corruption  │
  │    occurs — clean failure, not silent data corruption        │
  │                                                              │
  │  ✓ Efficient split context                                   │
  │    Function calls: save Upper only (16 words)                │
  │    Interrupts: save Lower only (16 words)                    │
  │    → 50% less memory traffic vs saving all registers         │
  │                                                              │
  │  ✓ Hardware-linked chain is ISO 26262 friendly               │
  │    Saved context is always in a known, hardware-defined      │
  │    location — easier to verify for safety analysis           │
  │                                                              │
  │  ✓ Nested interrupts naturally supported                     │
  │    Each nesting level consumes exactly one CSA — linear,     │
  │    predictable resource usage                                │
  └──────────────────────────────────────────────────────────────┘
```

---

## 22 · CSA vs ARM Cortex-M Stack

| Feature | ARM Cortex-M | TriCore CSA |
|---|---|---|
| **Context storage location** | Main stack (or process stack) | Dedicated CSA pool in SRAM |
| **Save mechanism** | HW saves 8 regs; SW saves rest | HW saves all — no SW involvement |
| **PUSH/POP instructions** | Yes — ISR prologue/epilogue | No — hardware manages entirely |
| **Interrupt latency** | 12–16+ cycles (HW + SW save) | ~5 cycles (hardware only) |
| **Latency determinism** | Depends on SW save code path | Fixed, hardware-defined |
| **Nested interrupt cost** | Stack grows per nesting level | 1 CSA per level (fixed 64 bytes) |
| **Stack overflow** | Silent memory corruption possible | Trap before any corruption |
| **Worst-case analysis** | Requires analyzing SW prologues | Hardware-defined, trivially provable |
| **Context split** | All registers saved on interrupt | Upper (calls) / Lower (IRQ) split |
| **Memory efficiency** | Must size stack for max nesting | CSA pool sized to max depth only |
| **Debug visibility** | Stack unwind via DWARF info | CSA chain directly readable via PCXI |
| **Safety certification** | Stack analysis is complex | CSA chain hardware-verified |

> **Bottom line:** For a brake-by-wire or steering control system where worst-case interrupt
> latency **must be proven** for ISO 26262 certification, the CSA mechanism is not just
> better — it is the correct tool for the job.

---

## 23 · Summary

The Context Save Area is the architectural feature that most clearly defines TriCore as
an **automotive-purpose** CPU rather than a general-purpose embedded processor.

### The Complete Mental Model

```
  ┌──────────────────────────────────────────────────────────────┐
  │                  CSA Architecture Summary                    │
  │                                                              │
  │  CONCEPT              MEANING                               │
  │  ────────────────────────────────────────────────────────    │
  │  Context         =    Complete CPU snapshot (registers)      │
  │  CSA             =    Fixed 64-byte block holding 1 context  │
  │  CSA Pool        =    Pre-allocated SRAM region at startup   │
  │  FCX register    =    Points to first FREE CSA               │
  │  LCX register    =    Marks end of CSA pool (safety limit)   │
  │  PCXI register   =    Points to current ACTIVE context chain │
  │  Upper Context   =    Callee-saved regs (D8–15, A10–15)      │
  │                       Saved on: CALL instruction             │
  │  Lower Context   =    Caller-saved regs (D0–7, A2–7)         │
  │                       Saved on: Interrupt entry              │
  │                                                              │
  │  OPERATION SUMMARY                                           │
  │  ────────────────────────────────────────────────────────    │
  │  Function call   →  allocate CSA, save Upper, link chain     │
  │  Interrupt entry →  allocate CSA, save Lower, link chain     │
  │  Function return →  restore Upper from CSA, return to FCX    │
  │  Interrupt return→  restore Lower from CSA, return to FCX    │
  │  Pool exhaustion →  Trap Class 4 — before any corruption     │
  └──────────────────────────────────────────────────────────────┘
```

### Why This Matters in Automotive Safety

```
  Traditional stack → "I believe my stack is big enough"  (hope)
  TriCore CSA       → "The hardware will trap before corruption" (guarantee)

  Traditional stack → "Interrupt latency is ~N cycles" (approximate)
  TriCore CSA       → "Interrupt latency is exactly N cycles" (provable)

  ISO 26262 ASIL-D demands proofs, not approximations.
  The CSA architecture is how TriCore delivers them.
```

### What Comes Next in Part 4

| Topic | Why It Matters |
|---|---|
| **Interrupt Controller (ICU/IR)** | Priority arbitration, SRC registers, ICR |
| **Interrupt vector table** | How TriCore locates and dispatches ISR handlers |
| **BIV register** | Base Interrupt Vector — configuring the vector table location |
| **CPU Priority Numbers** | CCPN, PIPN — managing priority levels |
| **Interrupt latency calculation** | Proving worst-case response for ASIL |
| **Trap vector table** | Trap class routing to handler code |
| **OS integration** | How FreeRTOS and AUTOSAR OS use CSA for task switching |

---

*Infineon AURIX & TriCore Architecture Series — Part 3 of N*
*Based on publicly available Infineon TriCore Architecture Volume 1 (ISA) and Volume 2 (Core SFRs).*