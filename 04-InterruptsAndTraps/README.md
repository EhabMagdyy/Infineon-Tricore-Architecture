# TriCore Interrupt & Trap Architecture
### Part 4 — Service Request Nodes, Interrupt Routing, Priorities & Trap Handling

> AURIX contains one of the most flexible interrupt architectures in any automotive MCU.
> Rather than wiring peripherals directly to a CPU interrupt controller, every event
> passes through an **SRN (Service Request Node)** — giving you per-event priority,
> per-event CPU routing, and DMA offloading, all in hardware.

---

## Table of Contents

| # | Topic |
|---|---|
| 1 | [Introduction](#1--introduction) |
| 2 | [Interrupts vs Traps](#2--interrupts-vs-traps) |
| 3 | [Why TriCore Interrupts Are Different](#3--why-tricore-interrupts-are-different) |
| 4 | [Interrupt Architecture Overview](#4--interrupt-architecture-overview) |
| 5 | [Service Request Nodes (SRN)](#5--service-request-nodes-srn) |
| 6 | [Service Request Control Registers (SRC)](#6--service-request-control-registers-src) |
| 7 | [Interrupt Routing](#7--interrupt-routing) |
| 8 | [Type of Service (TOS)](#8--type-of-service-tos) |
| 9 | [Interrupt Priorities (SRPN)](#9--interrupt-priorities-srpn) |
| 10 | [Interrupt Processing Flow](#10--interrupt-processing-flow) |
| 11 | [Interrupt Vector Table](#11--interrupt-vector-table) |
| 12 | [Interrupt Service Routines (ISR)](#12--interrupt-service-routines-isr) |
| 13 | [Nested Interrupts](#13--nested-interrupts) |
| 14 | [Software Interrupts](#14--software-interrupts) |
| 15 | [DMA as Interrupt Target](#15--dma-as-interrupt-target) |
| 16 | [Multi-Core Interrupt Routing](#16--multi-core-interrupt-routing) |
| 17 | [Trap Architecture](#17--trap-architecture) |
| 18 | [Trap Classes](#18--trap-classes) |
| 19 | [Trap Vector Table](#19--trap-vector-table) |
| 20 | [Trap Handling Examples](#20--trap-handling-examples) |
| 21 | [Interrupt vs Trap Comparison](#21--interrupt-vs-trap-comparison) |
| 22 | [Automotive Use Cases](#22--automotive-use-cases) |
| 23 | [Summary](#23--summary) |

---

## 1 · Introduction

Most embedded MCUs route peripheral interrupts directly into a centralized interrupt
controller (e.g., ARM's NVIC). AURIX takes a fundamentally different approach — every
interrupt source owns its own **Service Request Node (SRN)**, an independent,
programmable interrupt object.

```
  Traditional MCU (ARM Cortex-M):          AURIX TriCore:
  ───────────────────────────────          ──────────────────────────────────
  Peripheral A ──┐                         Peripheral A ──► SRN_A ──┐
  Peripheral B ──┤                         Peripheral B ──► SRN_B ──┤
  Peripheral C ──┼──► NVIC ──► CPU         Peripheral C ──► SRN_C ──┤──► IR ──► CPUx
  Peripheral D ──┤                         Peripheral D ──► SRN_D ──┤         or DMA
  Peripheral E ──┘                         Peripheral E ──► SRN_E ──┘

  One central controller                   Per-event routing, priority,
  handles everything                       CPU target, and DMA option
```

This architecture enables:

| Capability | What It Means |
|---|---|
| **Per-event priority** | CAN RX and CAN Error get independent SRPN values |
| **Per-event CPU routing** | CAN packets go to CPU1, brake events to CPU0 |
| **DMA integration** | ADC results written to RAM without CPU involvement |
| **Multi-core scalability** | Each of 6 CPUs receives only its designated events |
| **Software-triggered IRQs** | Any CPU can trigger any SRN in software |

---

## 2 · Interrupts vs Traps

Both transfer control to a handler — but they originate from completely different
sources and serve different purposes.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    Interrupt                                 │
  │                                                              │
  │  Source:    External hardware peripheral                     │
  │  Timing:    Asynchronous — arrives at any instruction        │
  │  Routing:   Through SRN → Interrupt Router → CPU            │
  │  Purpose:   Notify CPU of a hardware event                   │
  │                                                              │
  │  Examples:                                                   │
  │  ├── CAN frame received                                      │
  │  ├── ADC conversion complete                                 │
  │  ├── Timer overflow                                          │
  │  └── Ethernet packet arrived                                 │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │                      Trap                                    │
  │                                                              │
  │  Source:    CPU itself (internal exception condition)        │
  │  Timing:    Synchronous — tied to a specific instruction     │
  │  Routing:   Through Trap Class → Trap Vector Table → Handler │
  │  Purpose:   Signal an illegal or exceptional CPU condition   │
  │                                                              │
  │  Examples:                                                   │
  │  ├── Divide by zero                                          │
  │  ├── Illegal / undefined opcode                              │
  │  ├── Memory protection violation (MPU)                       │
  │  └── CSA pool exhausted                                      │
  └──────────────────────────────────────────────────────────────┘
```

> **Key rule:** Interrupts come *to* the CPU from the outside world.
> Traps come *from* the CPU's own execution unit.

---

## 3 · Why TriCore Interrupts Are Different

### ARM Cortex-M (NVIC model)

```
  Peripheral
      │
      ▼
   NVIC  ←── Single central controller
      │       Priority arbitration here
      ▼       Fixed routing to one CPU
   CPU0
      │
      ▼
    ISR
```

### TriCore (SRN model)

```
  Peripheral Event
        │
        ▼
  ┌───────────┐
  │    SRN    │  ← Each event has its own node
  │           │    with its own priority (SRPN),
  │  SRPN     │    enable bit (SRE), pending flag (SRR),
  │  TOS      │    and CPU target (TOS)
  │  SRE/SRR  │
  └─────┬─────┘
        │
        ▼
  ┌───────────────┐
  │  Interrupt    │  ← Hardware priority arbiter
  │  Router (IR)  │    evaluates all pending SRNs
  └───────┬───────┘
          │
    ┌─────┴──────┐
    │            │
    ▼            ▼
  CPUx         DMA    ← Target chosen by TOS field
    │
    ▼
   ISR (via interrupt vector table)
```

The **SRN** is the fundamental unit of interrupt management in AURIX — everything
flows through it.

---

## 4 · Interrupt Architecture Overview

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                  AURIX Interrupt Architecture                    │
  │                                                                  │
  │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐      │
  │  │ CAN 0    │   │ ADC 0    │   │ GTM TIM0 │   │ ETH 0    │ ...  │
  │  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘      │
  │       │              │              │              │             │
  │  ┌────▼─────┐   ┌────▼─────┐   ┌───▼──────┐  ┌────▼─────┐       │
  │  │ SRN ×4   │   │ SRN ×4   │   │ SRN ×N   │  │ SRN ×N   │       │
  │  └────┬─────┘   └────┬─────┘   └───┬──────┘  └────┬─────┘       │
  │       │              │              │              │             │
  │       └──────────────┴──────────────┴──────────────┘             │
  │                                    │                             │
  │                         ┌──────────▼──────────┐                  │
  │                         │   Interrupt Router   │                 │
  │                         │   (Priority arbiter) │                 │
  │                         └──────────┬───────────┘                 │
  │                                    │                             │
  │               ┌────────────────────┼────────────────────┐        │
  │               │          │         │         │          │        │
  │               ▼          ▼         ▼         ▼          ▼        │
  │            CPU0       CPU1      CPU2      CPU3         DMA       │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 5 · Service Request Nodes (SRN)

An **SRN** is the intelligent interrupt object that sits between a peripheral event and
the CPU. Think of it as a **programmable interrupt ticket** — it describes what happened,
how urgent it is, and who should handle it.

### What an SRN Represents

```
  ┌─────────────────────────────────────────────────────────────┐
  │                  Service Request Node (SRN)                 │
  │                                                             │
  │  Field    │  Meaning                                        │
  │  ─────────┼──────────────────────────────────────────────── │
  │  SRPN     │  Priority number (0–255, higher = more urgent)  │
  │  TOS      │  Target: CPU0, CPU1, ..., CPU5, or DMA          │
  │  SRE      │  Enable bit (1 = interrupt enabled)             │
  │  SRR      │  Request pending flag (set by hardware)         │
  │  CLRR     │  Clear request (write 1 to acknowledge)         │
  │  SETR     │  Software set (write 1 to trigger in software)  │
  └─────────────────────────────────────────────────────────────┘
```

### SRNs per Peripheral — CAN Example

```
  ┌──────────────────────────────────────────────────────────┐
  │                    CAN0 Module                           │
  │                                                          │
  │  Event                SRN              Typical Priority  │
  │  ──────────────────────────────────────────────────────  │
  │  Frame received  ──► SRC_CAN0RXF0   → SRPN = 120        │
  │  Frame sent      ──► SRC_CAN0TX     → SRPN =  80        │
  │  Error passive   ──► SRC_CAN0ERR    → SRPN = 200        │
  │  Bus-off         ──► SRC_CAN0BOFF   → SRPN = 240        │
  │  FIFO full       ──► SRC_CAN0FIFO   → SRPN =  60        │
  └──────────────────────────────────────────────────────────┘
```

### SRNs per Peripheral — ADC Example

```
  ┌──────────────────────────────────────────────────────────┐
  │                    ADC0 Module (EDSADC/VADC)             │
  │                                                          │
  │  Event                SRN              Typical Priority  │
  │  ──────────────────────────────────────────────────────  │
  │  Conversion done ──► SRC_VADC0SRN0  → SRPN = 100        │
  │  Queue empty     ──► SRC_VADC0SRN1  → SRPN =  40        │
  │  Limit check     ──► SRC_VADC0SRN2  → SRPN = 180        │
  │  Result ready    ──► SRC_VADC0SRN3  → SRPN =  60        │
  └──────────────────────────────────────────────────────────┘
```

> Every single event can have a **completely independent** priority, CPU target, and
> enable state. This is fundamentally more flexible than NVIC where a peripheral shares
> one or a few IRQ lines.

---

## 6 · Service Request Control Registers (SRC)

Each SRN is configured through its **SRC register** — a memory-mapped 32-bit register
with a standardized layout across all AURIX peripherals.

### SRC Register Bit Layout

```
  SRC Register (32-bit):

  Bit  31          │  Reserved
  Bit  30          │  IOVCLR   — Interrupt overflow clear
  Bit  29          │  IOV      — Interrupt overflow flag
  Bit  28          │  ISRPR   — In-service request priority (read)
  Bits 27:24       │  Reserved
  Bit  23          │  SWS      — Software sticky bit
  Bit  22          │  SWSCLR  — Clear software sticky
  Bit  21          │  Reserved
  Bit  20          │  CLRR    — Clear request (write 1 to ack)
  Bit  19          │  SETR    — Software set request (write 1)
  Bit  18          │  RR      — Alias read of SRR
  Bit  17          │  SRR     — Service request pending (HW sets)
  Bit  16          │  SRE     — Service request enable
  Bits 15:13       │  TOS     — Type of Service (target CPU/DMA)
  Bits 12:11       │  Reserved
  Bits 10:8        │  Reserved
  Bits  7:0        │  SRPN    — Service Request Priority Number
```

### Common SRC Register Names

```
  SRC_CAN0RXF0      →  CAN0 RX FIFO 0 interrupt
  SRC_CAN0BOFF      →  CAN0 Bus-off error
  SRC_ASCLIN0TX     →  ASCLIN0 (UART) transmit
  SRC_ASCLIN0RX     →  ASCLIN0 receive
  SRC_GTMTIM00      →  GTM TIM0 channel 0
  SRC_VADC0SRN0     →  VADC group 0 result 0
  SRC_CCU60SR0      →  CCU60 service request 0
  SRC_STM0SR0       →  STM0 compare match 0
```

---

## 7 · Interrupt Routing

Interrupt routing is the process by which the Interrupt Router determines **which CPU**
(or DMA) receives a pending service request.

```
  Multiple SRNs pending simultaneously:

  SRN_A:  SRPN=120, TOS=CPU0, SRR=1
  SRN_B:  SRPN=200, TOS=CPU1, SRR=1
  SRN_C:  SRPN= 80, TOS=CPU0, SRR=1
  SRN_D:  SRPN=240, TOS=CPU2, SRR=1

  Interrupt Router decision (per CPU):
  ──────────────────────────────────────────────────────
  CPU0 sees:  SRN_A (120) and SRN_C (80)
              → selects SRN_A (highest for CPU0)

  CPU1 sees:  SRN_B (200)
              → selects SRN_B

  CPU2 sees:  SRN_D (240)
              → selects SRN_D

  Result:
  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │  CPU0   │    │  CPU1   │    │  CPU2   │
  │ ISR@120 │    │ ISR@200 │    │ ISR@240 │
  └─────────┘    └─────────┘    └─────────┘
  All three CPUs can be serving interrupts simultaneously!
```

---

## 8 · Type of Service (TOS)

The **TOS field** in the SRC register is the routing selector — it answers:
*"When this event fires, who handles it?"*

```
  TOS field encoding (3-bit):

  ┌────────────────────────────────────────────────┐
  │  TOS value │  Request routed to                │
  ├────────────────────────────────────────────────┤
  │  000       │  CPU0                             │
  │  001       │  CPU1                             │
  │  010       │  CPU2                             │
  │  011       │  CPU3                             │
  │  100       │  CPU4                             │
  │  101       │  CPU5                             │
  │  110       │  DMA (no CPU involvement)         │
  │  111       │  Reserved                         │
  └────────────────────────────────────────────────┘
```

### TOS Routing Example

```
  Motor control system with 3 CPUs:

  SRC_CCU60SR0   → TOS = CPU0  (motor PWM ISR on safety core)
  SRC_CAN0RXF0   → TOS = CPU1  (CAN receive on comms core)
  SRC_VADC0SRN0  → TOS = CPU0  (ADC result on control core)
  SRC_ASCLIN0RX  → TOS = CPU2  (UART debug on diagnostics core)
  SRC_VADC1SRN0  → TOS = DMA   (second ADC channel → DMA → RAM, CPU never wakes)

  Result:
  ──────────────────────────────────────────────────────────
  CPU0:  Handles motor control and ADC — safety functions
  CPU1:  Handles CAN communication — networking functions
  CPU2:  Handles UART — diagnostics/debug functions
  DMA:   Handles second ADC channel — no CPU load at all
```

---

## 9 · Interrupt Priorities (SRPN)

The **SRPN (Service Request Priority Number)** is an 8-bit value (0–255) that determines
which pending interrupt is served first when multiple events are waiting for the same CPU.

```
  SRPN range:  0 (lowest) ──────────────────────► 255 (highest)

  Example priority assignments for an EV motor controller:

  ┌─────────────────────────────────────────────────────┐
  │  SRPN  │  Interrupt Source         │  Reason        │
  ├─────────────────────────────────────────────────────┤
  │  255   │  SMU safety alarm         │  Critical fault│
  │  250   │  Brake pressure ISR       │  ASIL-D safety │
  │  240   │  Motor overcurrent        │  Hardware prot │
  │  220   │  Motor control PWM sync   │  Real-time ctrl│
  │  180   │  ADC limit check          │  Threshold mon │
  │  120   │  CAN RX frame             │  Comms         │
  │   80   │  CAN TX complete          │  Comms         │
  │   40   │  UART diagnostic RX       │  Debug         │
  │   20   │  NVM write complete       │  Background    │
  │    1   │  Low-priority housekeep.  │  Background    │
  └─────────────────────────────────────────────────────┘
```

### Priority Preemption Rule

```
  Currently executing: CAN_ISR()  at SRPN = 120
                            │
  New event: BRAKE_ISR     at SRPN = 250
                            │
                            ▼
  250 > 120 → BRAKE_ISR preempts CAN_ISR

  Execution:
  ─────────────────────────────────────────►  time
  ▓▓▓▓▓▓▓CAN_ISR▓▓▓[preempted]░░░░BRAKE_ISR░░░[return]▓▓▓▓CAN_ISR▓▓▓
```

> Priority 0 (SRPN=0) means the SRN is **disabled** — no interrupt will be routed
> regardless of the SRE bit.

---

## 10 · Interrupt Processing Flow

Step-by-step trace from a hardware event to ISR execution and return:

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  Step 1 │  Hardware Event                                        │
  │          │  CAN controller receives a valid CAN frame            │
  └────────────────────────────────┬─────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  Step 2 │  SRN Sets Pending Flag                                 │
  │          │  SRC_CAN0RXF0.SRR = 1  (hardware sets this)          │
  └────────────────────────────────┬─────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  Step 3 │  SRE Check                                             │
  │          │  Is SRC_CAN0RXF0.SRE == 1?  (interrupt enabled?)     │
  │          │  No  → request ignored                                │
  │          │  Yes → request forwarded to Interrupt Router          │
  └────────────────────────────────┬─────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  Step 4 │  Priority Arbitration                                  │
  │          │  Interrupt Router compares SRPN against current       │
  │          │  CPU priority (ICR.CCPN)                              │
  │          │  New SRPN > CCPN?  → interrupt is accepted            │
  └────────────────────────────────┬─────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  Step 5 │  Context Save (CSA)                                    │
  │          │  Hardware saves Lower Context into a CSA              │
  │          │  (see Part 3 — no software PUSH involved)             │
  └────────────────────────────────┬─────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  Step 6 │  Vector Lookup                                         │
  │          │  CPU calculates ISR address:                          │
  │          │  address = BIV + (SRPN × 0x20)  (8-byte entries)     │
  └────────────────────────────────┬─────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  Step 7 │  ISR Executes                                          │
  │          │  CanRxISR() runs                                      │
  │          │  ISR clears SRR:  SRC_CAN0RXF0.CLRR = 1             │
  │          │  Reads CAN data from peripheral registers             │
  └────────────────────────────────┬─────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  Step 8 │  RFE (Return from Exception)                           │
  │          │  Hardware restores Lower Context from CSA             │
  │          │  ICR.CCPN restored to previous priority               │
  │          │  CPU resumes interrupted task at exact instruction     │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 11 · Interrupt Vector Table

The **interrupt vector table** maps each SRPN to an ISR handler address. Its location
in Flash is configured by the **BIV (Base Interrupt Vector) register**.

```
  BIV register:  0x8010_0000  (example, configured at startup)

  Vector table layout (each entry = 32 bytes = 0x20):

  Address              SRPN   Handler
  ────────────────────────────────────────────────────
  0x8010_0000 + 0×20 =  0    (unused / priority 0)
  0x8010_0000 + 1×20 =  1    LowPrio_ISR
  ...
  0x8010_0000 + 20×20 = 20   UART_RX_ISR
  ...
  0x8010_0000 + 80×20 = 80   CAN_TX_ISR
  0x8010_0000 + 120×20= 120  CAN_RX_ISR       ◄── CAN RX handler
  ...
  0x8010_0000 + 220×20= 220  MotorControl_ISR
  0x8010_0000 + 250×20= 250  Brake_ISR
  0x8010_0000 + 255×20= 255  SMU_Alarm_ISR
  ────────────────────────────────────────────────────
  Each 32-byte slot holds the ISR jump instruction
  (typically a JA or BISR opcode)
```

> The table is **indexed directly by SRPN** — the vector lookup is a single multiply
> and add, making it O(1) with no search or indirection.

---

## 12 · Interrupt Service Routines (ISR)

### ISR Declaration — iLLD / HighTec GCC Style

```c
/* IFX_INTERRUPT macro places ISR into the vector table automatically */
IFX_INTERRUPT(CAN0_RX_ISR, 0, 120)
{
    /* Arguments:
     *   CAN0_RX_ISR  = ISR function name
     *   0            = Target CPU (CPU0)
     *   120          = SRPN (priority)
     */

    /* Step 1: Acknowledge the interrupt */
    SRC_CAN0RXF0.B.CLRR = 1;

    /* Step 2: Read received data from CAN peripheral */
    IfxCan_Node_readMessage(&can0Node, &rxMsg);

    /* Step 3: Post message to application queue */
    postCanMessage(&rxMsg);
}
```

### SRC Register Initialization

```c
/* Configure the SRN before enabling the interrupt */
void initCAN0_RX_Interrupt(void)
{
    /* Set priority */
    SRC_CAN0RXF0.B.SRPN = 120;

    /* Route to CPU0 */
    SRC_CAN0RXF0.B.TOS  = 0;

    /* Enable the interrupt */
    SRC_CAN0RXF0.B.SRE  = 1;
}
```

---

## 13 · Nested Interrupts

TriCore naturally supports nested interrupts through the CSA chain (see Part 3).
A higher-SRPN interrupt **always preempts** a lower-SRPN ISR running on the same CPU.

```
  Timeline — CAN_ISR (SRPN=120) preempted by BRAKE_ISR (SRPN=250):

  ──────────────────────────────────────────────────────────►  time
  │      Task (SRPN=0)       │
  │                          │  CAN RX fires (120 > 0) ──────►│
  │                          │  Context save (CSA)             │
  │                          │  CAN_ISR begins ───────────────►│
  │                          │                                 │  BRAKE fires (250>120)
  │                          │                                 │  Context save (CSA)
  │                          │                                 │  BRAKE_ISR begins ────►│
  │                          │                                 │                        │
  │                          │                                 │                        │ BRAKE_ISR done
  │                          │                                 │  ◄── Context restore   │
  │                          │  CAN_ISR resumes ◄──────────────│
  │                          │  CAN_ISR done                   │
  │  Task resumes ◄──────────│
```

### CSA Chain During Nesting (Recap)

```
  PCX
   │
   ▼
  ┌──────────────────────┐
  │  BRAKE_ISR  Lower Ctx│  ← Currently executing
  └──────────┬───────────┘
             │
   ▼
  ┌──────────────────────┐
  │  CAN_ISR    Lower Ctx│  ← Suspended
  └──────────┬───────────┘
             │
   ▼
  ┌──────────────────────┐
  │  Task       Upper Ctx│  ← Original task context
  └──────────────────────┘
```

---

## 14 · Software Interrupts

Any CPU can trigger any SRN in software by writing to the **SETR bit** of the target
SRC register.

```c
/* CPU0 triggers a software interrupt on CPU1 at SRPN = 50 */
SRC_GPSR0_SR0.B.SRPN = 50;   /* General Purpose SRN */
SRC_GPSR0_SR0.B.TOS  = 1;   /* Route to CPU1       */
SRC_GPSR0_SR0.B.SRE  = 1;   /* Enable              */
SRC_GPSR0_SR0.B.SETR = 1;   /* Fire! (software set) */
```

### Software Interrupt Use Cases

```
  ┌────────────────────────────────────────────────────────────┐
  │  Use Case                │  How                           │
  ├────────────────────────────────────────────────────────────┤
  │  Inter-core signaling    │  CPU0 signals CPU1 via GPSR SRN │
  │  (IPI equivalent)        │  after writing to shared LMU    │
  ├────────────────────────────────────────────────────────────┤
  │  ISR unit testing        │  Trigger any ISR in software    │
  │                          │  without real hardware event    │
  ├────────────────────────────────────────────────────────────┤
  │  OS task activation      │  AUTOSAR OS uses SW IRQ to      │
  │                          │  activate a task on another CPU │
  └────────────────────────────────────────────────────────────┘
```

---

## 15 · DMA as Interrupt Target

Setting `TOS = DMA` routes a peripheral event directly to the DMA controller —
bypassing the CPU entirely.

### CPU-Driven ADC Transfer (Traditional)

```
  ADC completes conversion
          │
          ▼
   CPU0 wakes (interrupt)
          │
          ▼
   CPU0 reads ADC result register
          │
          ▼
   CPU0 writes result to RAM buffer
          │
          ▼
   CPU0 returns to task

  Cost: Interrupt overhead + CPU cycles for every sample
```

### DMA-Driven ADC Transfer

```
  ADC completes conversion
          │
          ▼  TOS = DMA
   DMA controller wakes
          │
          ▼
   DMA reads ADC result register
          │
          ▼
   DMA writes result to RAM buffer
          │
          ▼
   (CPU0 never interrupted)

  Cost: Zero CPU cycles per sample
```

### When to Use DMA Routing

```
  ✓ High-sample-rate ADC (e.g., 3-phase motor current at 10 kHz)
  ✓ SPI sensor burst reads (IMU, pressure sensor arrays)
  ✓ UART/ASCLIN bulk data reception
  ✓ Any scenario where CPU overhead per transfer is unacceptable
```

---

## 16 · Multi-Core Interrupt Routing

In a multi-core AURIX system, interrupt routing is the primary mechanism for **workload
partitioning** — each CPU receives only the interrupts relevant to its role.

### Example: 4-Core EV Controller

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    AURIX TC39x — Core Assignments               │
  │                                                                 │
  │  CPU0 (Safety Core)       CPU1 (Comms Core)                     │
  │  ─────────────────────    ──────────────────────                │
  │  SRC_CCU60SR0  → CPU0     SRC_CAN0RXF0   → CPU1                │
  │  SRC_VADC0SRN0 → CPU0     SRC_CAN1RXF0   → CPU1                │
  │  SRC_STM0SR0   → CPU0     SRC_ETH0SR0    → CPU1                │
  │  (Motor control, ADC,     (CAN, Ethernet                        │
  │   timing, braking)         gateway)                             │
  │                                                                 │
  │  CPU2 (App Core)          CPU3 (Diagnostics)                    │
  │  ─────────────────────    ──────────────────────                │
  │  SRC_GTMTIM00  → CPU2     SRC_ASCLIN0RX  → CPU3                │
  │  SRC_SENT0SR0  → CPU2     SRC_SMU0SR0    → CPU3                │
  │  (ADAS, sensor fusion,    (UDS diagnostics,                     │
  │   radar data)              safety alarms)                       │
  └─────────────────────────────────────────────────────────────────┘
```

### Benefit: True Parallel Interrupt Handling

```
  At the same moment in time:
  CPU0  ─── serving motor control ISR  (SRPN 220)
  CPU1  ─── serving CAN RX ISR         (SRPN 120)
  CPU2  ─── serving radar data ISR     (SRPN 160)
  CPU3  ─── serving UDS diagnostic ISR (SRPN  80)

  All four happening simultaneously — no arbitration needed
  because they target different CPUs.
```

---

## 17 · Trap Architecture

While interrupts are **asynchronous external events**, traps are **synchronous internal
exceptions** — triggered directly by the CPU when it encounters an illegal or boundary
condition during instruction execution.

```
  Interrupt path:                  Trap path:
  ──────────────                   ──────────
  External world                   CPU itself
        │                               │
  Peripheral event                 Instruction executes
        │                               │
       SRN                        Exception condition detected
        │                               │
  Interrupt Router                 Trap class determined
        │                               │
      CPUx ISR                    Trap vector table lookup
                                        │
                                   Trap handler executes
```

---

## 18 · Trap Classes

TriCore defines **8 trap classes**, each covering a category of CPU fault:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    TriCore Trap Classes                         │
  │                                                                 │
  │  Class │ Name                    │ Example Triggers            │
  │  ───────────────────────────────────────────────────────────── │
  │    0   │ MMU / Address Fault     │ Address translation error   │
  │        │                         │ (if MMU present)            │
  │  ───────────────────────────────────────────────────────────── │
  │    1   │ Internal Protection     │ MPU violation               │
  │        │                         │ Privilege access violation   │
  │        │                         │ ENDINIT register write      │
  │  ───────────────────────────────────────────────────────────── │
  │    2   │ Instruction Error       │ Undefined / illegal opcode  │
  │        │                         │ Invalid operand encoding     │
  │  ───────────────────────────────────────────────────────────── │
  │    3   │ Context Management      │ CSA free list empty (FCX≈LCX)│
  │        │                         │ CSA list corruption          │
  │  ───────────────────────────────────────────────────────────── │
  │    4   │ System Bus Error        │ Bus timeout                  │
  │        │                         │ Failed peripheral access     │
  │        │                         │ ECC double-bit error on bus  │
  │  ───────────────────────────────────────────────────────────── │
  │    5   │ Assertion Trap          │ Hardware consistency check   │
  │        │                         │ Debug assertion              │
  │  ───────────────────────────────────────────────────────────── │
  │    6   │ System Call (SYSCALL)   │ Explicit SYSCALL instruction │
  │        │                         │ Used by AUTOSAR OS for       │
  │        │                         │ supervisor-mode services     │
  │  ───────────────────────────────────────────────────────────── │
  │    7   │ NMI (Non-Maskable IRQ)  │ Critical safety fault        │
  │        │                         │ SMU-triggered fatal error    │
  │        │                         │ Cannot be masked or disabled │
  └─────────────────────────────────────────────────────────────────┘
```

### Trap Identification Number (TIN)

Within each class, a **TIN (Trap Identification Number)** is loaded into `D15` when the
trap fires, identifying the **exact sub-cause**:

```
  Class 1 (Internal Protection) TINs:
  TIN 1 → Privilege violation (user mode accessing supervisor resource)
  TIN 2 → Memory protection read violation
  TIN 3 → Memory protection write violation
  TIN 4 → Memory protection execute violation
  TIN 5 → Global register write protection

  Class 3 (Context Management) TINs:
  TIN 1 → Free CSA list depleted (FCX reached LCX warning point)
  TIN 2 → CSA list corruption detected
  TIN 3 → Free CSA list empty (FCX = NULL)
```

---

## 19 · Trap Vector Table

Separate from the interrupt vector table, the **trap vector table** maps each trap class
to a handler. Its location is set by the **BTV (Base Trap Vector) register**.

```
  BTV register:  0x8000_0100  (example)

  Trap vector table layout (each entry = 32 bytes):

  Address                Entry    Handler
  ────────────────────────────────────────────────
  BTV + 0×20 = 0x8000_0100  Class 0  MMU_TrapHandler
  BTV + 1×20 = 0x8000_0120  Class 1  Protection_TrapHandler
  BTV + 2×20 = 0x8000_0140  Class 2  InstrError_TrapHandler
  BTV + 3×20 = 0x8000_0160  Class 3  Context_TrapHandler
  BTV + 4×20 = 0x8000_0180  Class 4  BusError_TrapHandler
  BTV + 5×20 = 0x8000_01A0  Class 5  Assert_TrapHandler
  BTV + 6×20 = 0x8000_01C0  Class 6  Syscall_Handler
  BTV + 7×20 = 0x8000_01E0  Class 7  NMI_Handler
```

### Trap Entry Flow

```
  Trap condition detected during instruction execution
                      │
                      ▼
  CPU loads trap class → D15 = TIN (sub-cause code)
                      │
                      ▼
  Hardware saves context to CSA (same as interrupt)
                      │
                      ▼
  PC = BTV + (TrapClass × 0x20)
                      │
                      ▼
  Trap handler executes
                      │
              ┌───────┴────────┐
              │                │
         Recoverable       Not Recoverable
              │                │
         RFE (return)     Reset / Safe State
```

---

## 20 · Trap Handling Examples

### Example 1 — Divide by Zero

```c
int x = 10;
int y = 0;
int z = x / y;   /* ← DIV instruction with divisor = 0 */
```

```
  CPU executes DIV instruction
        │
        ▼
  Divisor = 0 detected by arithmetic unit
        │
        ▼
  Class 2 Trap (Instruction Error)
  TIN = 0x28  (arithmetic trap: divide by zero)
        │
        ▼
  D15 = 0x28
  Context saved to CSA
  PC → BTV + 2×0x20  (Class 2 handler)
        │
        ▼
  InstrError_TrapHandler():
    read D15  →  0x28 (divide by zero)
    log fault
    trigger safe state / reset
```

### Example 2 — MPU Memory Protection Violation

```c
/* Task A (user mode) tries to write to Task B's private region */
volatile uint32_t *ptr = (uint32_t*)0xD001_0000;  /* Task B's RAM */
*ptr = 42;  /* ← Memory protection violation */
```

```
  MPU detects write to protected region
        │
        ▼
  Class 1 Trap (Internal Protection)
  TIN = 3  (write protection violation)
        │
        ▼
  D15 = 3
  Trap PC = address of violating instruction
  Protection_TrapHandler():
    identify faulting task
    log: task name, address, violation type
    terminate faulting task (AUTOSAR OS)
    continue other tasks if possible
```

### Example 3 — CSA Pool Exhaustion

```
  Scenario: Infinite recursion exhausts the CSA free list

  void runaway(void) { runaway(); }  /* infinite recursion */

  Each CALL consumes 1 CSA (Upper Context save)
        │
        ▼
  FCX approaches LCX boundary
        │
        ▼
  Class 3 Trap, TIN 1 (CSA depletion warning — 1 CSA left)
        │
        ▼
  Context_TrapHandler():
    1 CSA remaining — cannot continue safely
    log the fault
    force system reset
    enter safe state
```

### Example 4 — SYSCALL (OS Service Request)

```c
/* User-mode task requests OS service (privilege escalation) */
__syscall(OS_SERVICE_GET_SEMAPHORE);   /* SYSCALL instruction */
```

```
  SYSCALL instruction executes
        │
        ▼
  Class 6 Trap
  D15 = OS_SERVICE_GET_SEMAPHORE  (service ID)
        │
        ▼
  Syscall_Handler():  (runs in supervisor mode)
    dispatch on D15
    execute OS service
    return result in D2
    RFE back to user task
```

---

## 21 · Interrupt vs Trap Comparison

| Feature | Interrupt | Trap |
|---|---|---|
| **Origin** | External hardware peripheral | CPU internal exception |
| **Timing** | Asynchronous (any instruction) | Synchronous (specific instruction) |
| **Routing mechanism** | SRN → Interrupt Router | Trap class → BTV |
| **Priority system** | SRPN (0–255) | Trap class (0–7) |
| **Vector table** | BIV + SRPN × 0x20 | BTV + Class × 0x20 |
| **Context save** | Hardware CSA (Lower Context) | Hardware CSA (same mechanism) |
| **Can be masked?** | Yes (SRE = 0, or ICR.IE = 0) | No (except Class 6/7 in some modes) |
| **Typical response** | Service the hardware event | Log + recover or reset |
| **Examples** | CAN RX, ADC done, Timer OVF | Div/0, illegal opcode, MPU fault |
| **Return instruction** | `RFE` | `RFE` (recoverable) or reset |
| **ISO 26262 role** | Event-driven real-time control | Fault containment & safe state |

---

## 22 · Automotive Use Cases

The combined interrupt + trap architecture is what allows a **single AURIX** to
simultaneously handle safety-critical control, communication, diagnostics, and
fault management:

```
  ┌──────────────────────────────────────────────────────────────────┐
  │              Automotive System — Interrupt Distribution          │
  │                                                                  │
  │  SRPN 250  Brake pressure anomaly   → CPU0  ASIL-D safety ctrl  │
  │  SRPN 240  Motor overcurrent ISR    → CPU0  Hardware protection  │
  │  SRPN 220  Motor PWM sync ISR       → CPU0  Real-time control    │
  │  SRPN 200  Radar data ready         → CPU2  ADAS processing      │
  │  SRPN 180  Battery cell OV/UV alarm → CPU0  BMS protection       │
  │  SRPN 160  Ethernet frame RX        → CPU1  V2X communication    │
  │  SRPN 120  CAN FD frame received    → CPU1  Vehicle networking   │
  │  SRPN  80  UDS diagnostic request   → CPU3  OBD service          │
  │  SRPN  40  UART trace output        → DMA   Zero CPU cost        │
  │  SRPN  20  Flash write complete     → CPU3  NVM management       │
  │                                                                  │
  │              Trap Architecture — Fault Containment               │
  │                                                                  │
  │  Class 7 NMI        SMU fatal alarm  → immediate safe state     │
  │  Class 3 Context    CSA exhaustion   → log + reset              │
  │  Class 1 Protection MPU violation    → isolate faulting task    │
  │  Class 6 SYSCALL    OS API request   → privilege transition     │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 23 · Summary

### The Complete Mental Model

```
  ┌──────────────────────────────────────────────────────────────────┐
  │              Interrupt & Trap Architecture Summary               │
  │                                                                  │
  │  INTERRUPT SIDE                                                  │
  │  ─────────────────────────────────────────────────────────────   │
  │  Peripheral event  →  SRN sets SRR=1                            │
  │  SRN               →  holds SRPN, TOS, SRE, SRR, CLRR, SETR   │
  │  Interrupt Router  →  arbitrates by SRPN per CPU                │
  │  TOS               →  routes to CPU0..5 or DMA                  │
  │  SRPN              →  0 (off) to 255 (highest), higher wins     │
  │  Vector table      →  BIV + SRPN × 0x20 → ISR address          │
  │  Context save      →  hardware CSA (Lower Context, ~5 cycles)   │
  │  Return            →  RFE restores context from CSA             │
  │                                                                  │
  │  TRAP SIDE                                                       │
  │  ─────────────────────────────────────────────────────────────   │
  │  CPU fault         →  trap class 0–7 + TIN in D15              │
  │  Trap vector       →  BTV + Class × 0x20 → handler             │
  │  Context save      →  hardware CSA (same mechanism)            │
  │  Recovery          →  RFE if recoverable, reset if not          │
  │                                                                  │
  │  SHARED MECHANISM                                                │
  │  ─────────────────────────────────────────────────────────────   │
  │  Both interrupts and traps use CSA for context save/restore     │
  │  Both use BIV/BTV register to locate their vector tables        │
  │  Both are serviced with deterministic, hardware-managed latency │
  └──────────────────────────────────────────────────────────────────┘
```

### Why This Architecture Wins in Automotive

```
  NVIC (ARM):                     AURIX SRN:
  ─────────────────────────────   ────────────────────────────────────
  All peripherals share          Each event has own priority,
  a few IRQ lines         →      routing, and enable control

  One CPU gets all IRQs  →       Each CPU gets only its events

  CPU must poll to find   →      SRPN directly selects vector;
  which peripheral fired         no polling needed

  Stack overflow is       →      CSA pool depletion triggers
  silent corruption              a trap before any corruption
```

### What Comes Next in Part 5

| Topic | Why It Matters |
|---|---|
| **GTM (Generic Timer Module)** | The most complex AURIX peripheral — PWM generation, input capture, motor control timing |
| **CCU6 (Capture Compare Unit)** | 3-phase motor PWM, encoder interface |
| **VADC (Versatile ADC)** | Multi-group ADC, queue-based conversion, result handling |
| **STM (System Timer)** | Free-running 64-bit timer for RTOS tick and profiling |
| **Peripheral SRC wiring** | Connecting peripheral events to SRNs in practice |

---

*Infineon AURIX & TriCore Architecture Series — Part 4 of N*
*Based on publicly available Infineon AURIX TC3xx User Manual Volume 1 and TriCore ISA documentation.*