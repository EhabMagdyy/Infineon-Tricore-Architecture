# AURIX TriCore Safety Architecture (ASIL-D)

> A comprehensive guide to functional safety mechanisms in Infineon AURIX microcontrollers,
> targeting ISO 26262 ASIL-D compliance for automotive-grade embedded systems.

---

## Table of Contents

1. [Introduction to Functional Safety](#1-introduction-to-functional-safety)
2. [What is ASIL-D?](#2-what-is-asil-d)
3. [Why AURIX Needs Safety Architecture](#3-why-aurix-needs-safety-architecture)
4. [Safety Concept Overview](#4-safety-concept-overview)
5. [AURIX Safety Mechanisms](#5-aurix-safety-mechanisms)
   - 5.1 [Lockstep CPU](#51-lockstep-cpu)
   - 5.2 [Error Correction Code (ECC)](#52-error-correction-code-ecc)
   - 5.3 [Safety Management Unit (SMU)](#53-safety-management-unit-smu)
   - 5.4 [Watchdog System](#54-watchdog-system)
   - 5.5 [ENDINIT Protection](#55-endinit-protection)
   - 5.6 [Redundant Hardware](#56-redundant-hardware)
6. [Fault Detection and Reaction Flow](#6-fault-detection-and-reaction-flow)
7. [Memory Safety Architecture](#7-memory-safety-architecture)
8. [Multi-Core Safety Concept](#8-multi-core-safety-concept)
9. [ASIL-D Software Architecture](#9-asil-d-software-architecture)
10. [Automotive Example: EV Motor Controller](#10-automotive-example-ev-motor-controller)
11. [Summary](#11-summary)

---

## 1. Introduction to Functional Safety

Modern vehicles contain dozens of software-controlled systems where a single failure can
cause dangerous or life-threatening situations. Examples of safety-critical automotive systems:

| System | Risk if Failure Occurs |
|---|---|
| Brake control | Loss of braking ability |
| Steering systems | Loss of vehicle directional control |
| Battery management (BMS) | Thermal runaway, fire |
| Motor control | Unintended acceleration or deceleration |
| Airbag systems | Failure to deploy or unintended deployment |

### Normal vs. Safety-Critical Systems

A **standard embedded system** focuses on:

```
Correct Function
```

A **safety-critical embedded system** focuses on:

```
Correct Function
  +
Detecting Failures
  +
Reacting Safely
```

### The Core Safety Question

> _"What happens if something goes wrong?"_

Without a safety architecture, a CPU failure propagates through the system unchecked:

```
CPU Fault
    │
    ▼
Wrong Calculation
    │
    ▼
Wrong Motor Command
    │
    ▼
DANGER
```

A proper safety architecture **intercepts** this chain of failure before it reaches a
dangerous state.

---

## 2. What is ASIL-D?

**ASIL** stands for **Automotive Safety Integrity Level**, defined by the international
standard **ISO 26262** (_Road vehicles — Functional safety_).

ASIL specifies the required rigor of safety measures for a given system, based on:

- **Severity** of potential harm
- **Exposure** to the hazardous situation
- **Controllability** by the driver

### ASIL Levels

```
ASIL-A  ──  Lowest safety requirement
ASIL-B
ASIL-C
ASIL-D  ──  Highest automotive safety requirement
```

### ASIL-D Requirements

ASIL-D imposes the **strictest** constraints:

- Very low probability of dangerous systematic or random hardware failure
- Strong and comprehensive fault detection coverage
- Redundant safety mechanisms at hardware and software levels
- Deterministic, verified safe reaction to every detected fault

### Examples of ASIL-D Systems

- Brake-by-wire
- Electric power steering
- Powertrain / traction control
- Active safety systems (AEB, ESC)

---

## 3. Why AURIX Needs Safety Architecture

Automotive MCUs execute algorithms that directly command physical actuators.

### Example: Electric Motor Controller

```
         ┌─────────────────────────┐
         │         AURIX           │
         │                         │
Pedal ──►│  Motor Control          │
Position │  Algorithm              │──► PWM Output ──► Motor Torque
         │                         │
         └─────────────────────────┘
```

A silent CPU calculation error here produces a **wrong torque command**, which translates
directly to unintended vehicle movement — with no warning to the driver.

### The Consequence of No Safety Architecture

```
CPU Error (silent)
      │
      ▼
Wrong Algorithm Output
      │
      ▼
Wrong Actuator Command
      │
      ▼
Physical Danger
```

AURIX provides hardware and software mechanisms that **prevent silent failures** from
propagating to actuators.

---

## 4. Safety Concept Overview

AURIX follows a structured fault-handling philosophy with four phases:

```
    Fault Occurs
         │
         ▼
    ┌─────────┐
    │Detection│   ◄── Hardware comparators, ECC checkers, watchdogs
    └────┬────┘
         │
         ▼
    ┌─────────┐
    │Diagnosis│   ◄── SMU identifies the fault source and severity
    └────┬────┘
         │
         ▼
    ┌─────────┐
    │Reaction │   ◄── Reset, interrupt, or transition to safe state
    └────┬────┘
         │
         ▼
    ┌──────────┐
    │Safe State│   ◄── System is in a known, controlled, non-dangerous condition
    └──────────┘
```

### Example End-to-End

```
CPU Failure Detected
        │
        ▼
SMU Receives Alarm
        │
        ▼
Configured Reaction Triggered
        │
        ▼
System Reset / Safe Mode Entered
```

---

## 5. AURIX Safety Mechanisms

AURIX implements **multiple independent safety layers**. No single mechanism alone achieves
ASIL-D — the **combination** of all layers together provides the required integrity level.

```
┌─────────────────────────────────┐
│        Lockstep CPU             │  ◄── Processing redundancy
├─────────────────────────────────┤
│    ECC Memory Protection        │  ◄── Data integrity
├─────────────────────────────────┤
│       Watchdog System           │  ◄── Software liveness
├─────────────────────────────────┤
│   Safety Management Unit (SMU)  │  ◄── Central fault coordinator
├─────────────────────────────────┤
│     ENDINIT Protection          │  ◄── Register access control
├─────────────────────────────────┤
│     Redundant Hardware          │  ◄── Sensor / peripheral backup
└─────────────────────────────────┘
```

---

### 5.1 Lockstep CPU

Lockstep is one of the **most critical** AURIX safety features for CPU fault detection.

#### Concept

Two CPU cores execute **the same instruction stream simultaneously**. A hardware comparator
checks that both cores produce **identical results** on every cycle.

```
         Program Code
               │
     ┌─────────┴─────────┐
     │                   │
   CPU0               CPU1
  (Master)           (Checker)
  Execute            Execute
     │                   │
     └─────────┬─────────┘
               │
          Comparator
               │
         ┌─────┴──────┐
         │            │
       Match        Mismatch
         │            │
        OK          FAULT
                      │
                   SMU Alarm
```

#### Normal Operation

```
CPU0:  5 + 3 = 8
CPU1:  5 + 3 = 8
             ──────
Comparator:  MATCH → OK
```

#### Fault Detection

```
CPU0:  5 + 3 = 8
CPU1:  5 + 3 = 9   ← Hardware fault in CPU1 register
             ──────
Comparator:  MISMATCH → FAULT DETECTED → SMU Alarm
```

> The lockstep mechanism detects **any** computation divergence between the two cores,
> including stuck bits, flip errors, and transient faults due to radiation or EMI.

---

### 5.2 Error Correction Code (ECC)

Memory is vulnerable to bit flips caused by radiation, voltage transients, and hardware aging.
ECC adds **extra parity bits** alongside stored data to enable automatic detection and
correction.

#### How ECC Works

```
Write:
  Data: 10101100
  ECC:  [calculated parity bits]
  Stored: 10101100 | 101  (data + ECC bits)

Read:
  Memory → ECC Engine → Check parity
               │
       ┌───────┴────────┐
       │                │
  No Error        Error Detected
       │                │
   Return Data    ┌─────┴──────┐
                  │            │
            Single Bit    Double Bit
              Error          Error
                │               │
            Auto-Correct    Cannot Correct
            (transparent)       │
                            SMU Alarm
```

#### Single-Bit Error (Correctable)

```
Stored:  10101100
Read:    10100100  ← 1 bit flipped
ECC:     Detects and corrects automatically → 10101100
Result:  Transparent to application
```

#### Double-Bit Error (Detectable, Not Correctable)

```
Stored:  10101100
Read:    10100111  ← 2 bits flipped
ECC:     Detects error, cannot reconstruct original
Result:  SMU alarm triggered → safe reaction
```

---

### 5.3 Safety Management Unit (SMU)

The SMU is the **central safety controller** of AURIX. It acts as the single collection point
for all safety alarms from every subsystem.

#### Architecture

```
┌──────────┐  ┌──────────┐  ┌────────────┐
│   CPU    │  │  Memory  │  │ Peripheral │
│ (lockstep│  │  (ECC)   │  │  Monitors  │
│  error)  │  │  errors  │  │            │
└────┬─────┘  └────┬─────┘  └──────┬─────┘
     │              │               │
     └──────────────┼───────────────┘
                    │
               ┌────▼────┐
               │   SMU   │
               └────┬────┘
                    │
          ┌─────────┼──────────┐
          │         │          │
       Reset    Interrupt   Safe State
                            Output
```

#### Fault Sources Monitored by SMU

- CPU lockstep comparison errors
- ECC single and double-bit memory errors
- Watchdog timeout events
- Clock monitor failures (PLL loss of lock)
- Voltage monitor out-of-range
- Bus integrity errors
- Peripheral self-test failures

#### SMU Reaction Configuration

Each alarm source has a **configurable reaction**:

| Alarm Source | Possible Reactions |
|---|---|
| ECC Double-Bit Error | NMI, Reset, Safe State |
| Lockstep Mismatch | NMI, Reset, Safe State |
| Watchdog Timeout | Reset |
| Clock Failure | Reset, Safe Output |
| Voltage Out-of-Range | Reset, Safe Output |

---

### 5.4 Watchdog System

Watchdogs detect **software hangs** and infinite loops that would prevent the system from
performing its safety functions.

#### The Problem

```c
// Software gets stuck
while (condition_never_clears) {
    // CPU never exits
    // Safety tasks never execute
    // No actuator update
}
```

If software stops executing normally, the system may silently deliver **stale or wrong
outputs** — or no output at all.

#### The Solution: Watchdog Timer

```
Normal Operation:
  Safety Task Runs
        │
        ▼
  Refresh (kick) Watchdog  ← Must happen within timeout window
        │
        ▼
  Watchdog timer resets
        │
  (repeat)

Fault Condition:
  Software stuck / hanged
        │
        ▼
  No watchdog refresh
        │
        ▼
  Timeout expires
        │
        ▼
  Watchdog triggers reset
```

#### AURIX Watchdog Types

| Watchdog | Scope | Purpose |
|---|---|---|
| **Safety Watchdog (WDT_S)** | System-wide | Monitors overall system liveness |
| **CPU Watchdog (WDT_CPU0/1/2)** | Per-core | Monitors each CPU core independently |

> AURIX watchdogs also support a **windowed mode**: the software must kick the watchdog
> only within a specific time window — not too early, not too late — preventing a simple
> runaway loop from accidentally refreshing the watchdog.

---

### 5.5 ENDINIT Protection

ENDINIT is a **hardware lock** that protects critical system configuration registers from
being accidentally (or maliciously) modified by application software.

#### The Risk Without ENDINIT

```
Software Bug or Buffer Overflow
          │
          ▼
Overwrites Clock Configuration Register
          │
          ▼
System Clock Changes
          │
          ▼
Timing Violations, System Malfunction
```

#### ENDINIT Protection Flow

```
Normal operation: Register is LOCKED
          │
          ▼
Trusted init code: Unlock sequence (specific password write)
          │
          ▼
Modify protected register (clock, safety config, protection settings)
          │
          ▼
Re-lock immediately
          │
          ▼
Register locked again
```

Protected registers include:

- System clock and PLL configuration
- Safety register configurations
- Watchdog configuration
- Memory protection unit settings

---

### 5.6 Redundant Hardware

For the highest-confidence sensor and actuator paths, AURIX supports **hardware redundancy**
where critical signals are duplicated and cross-compared.

#### Dual Sensor Example

```
Physical Quantity (e.g., steering angle)
         │
    ┌────┴────┐
    │         │
Sensor A   Sensor B
    │         │
    └────┬────┘
         │
    Comparator
         │
   ┌─────┴──────┐
   │            │
 A ≈ B        A ≠ B
   │            │
  OK         FAULT
              │
         SMU Alarm
```

#### Normal Reading

```
Sensor A = 50°
Sensor B = 50°   ← Within tolerance
Result:    AGREE → Use value
```

#### Fault Detection

```
Sensor A = 50°
Sensor B = 120°  ← Outside tolerance
Result:    MISMATCH → Sensor failure detected → SMU alarm
```

---

## 6. Fault Detection and Reaction Flow

The complete fault handling sequence from initial detection to safe state:

```
        ┌──────────────────────────────┐
        │        Fault Occurs          │
        │  (HW failure, bit flip,      │
        │   software hang, sensor err) │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │         Detection            │
        │  Lockstep / ECC / Watchdog / │
        │  Clock monitor / Voltage mon │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │         SMU Alarm            │
        │  Alarm registered, severity  │
        │  class evaluated             │
        └──────────────┬───────────────┘
                       │
             ┌─────────┴──────────┐
             │                    │
             ▼                    ▼
     ┌───────────────┐   ┌──────────────────┐
     │  System Reset │   │  Safe Output /   │
     │  (hard reset) │   │  Safe State Mode │
     └───────────────┘   └──────────────────┘
```

> In a safe state, actuators are commanded to a **known safe position** (e.g., motor torque
> = 0, brakes = applied) rather than holding the last computed value.

---

## 7. Memory Safety Architecture

All memory regions in AURIX are protected by ECC to prevent corrupted code or data from
reaching the execution path.

```
┌──────────────────────────────────────┐
│             AURIX Memory             │
│                                      │
│  ┌─────────────┐   ┌──────────────┐  │
│  │    Flash    │   │     RAM      │  │
│  │  (Program)  │   │   (Data)     │  │
│  │    + ECC    │   │    + ECC     │  │
│  └──────┬──────┘   └──────┬───────┘  │
│         │                 │          │
│         ▼                 ▼          │
│  ┌─────────────┐   ┌──────────────┐  │
│  │   I-Cache   │   │   D-Cache    │  │
│  │    + ECC    │   │    + ECC     │  │
│  └─────────────┘   └──────────────┘  │
└──────────────────────────────────────┘
```

### Memory Safety Goals

| Goal | Mechanism |
|---|---|
| Prevent corrupted code execution | Flash ECC + cache ECC |
| Prevent wrong data usage | RAM ECC + data cache ECC |
| Detect silent memory degradation | Background ECC scrubbing |
| Isolate memory regions between tasks | Memory Protection Unit (MPU) |

---

## 8. Multi-Core Safety Concept

AURIX devices contain multiple TriCore CPU cores. Safety architectures leverage **core
isolation** to contain fault propagation.

### Core Role Assignment Example

```
┌─────────────────────────────────────────┐
│                  AURIX                  │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  CPU0    │  │  CPU1    │  │  CPU2  │ │
│  │          │  │          │  │        │ │
│  │ Safety   │  │ Comms /  │  │ Diag / │ │
│  │ Control  │  │ Gateway  │  │ Monitor│ │
│  └──────────┘  └──────────┘  └────────┘ │
└─────────────────────────────────────────┘
```

### Fault Isolation

CPU cores are isolated: a failure or runaway in CPU1 does **not** directly corrupt the
execution of the safety-critical function running on CPU0.

```
CPU1 Failure
      │
      ✗  (Does NOT propagate)
      │
CPU0 Safety Function continues executing
```

### Inter-Core Communication

Safe inter-core data exchange uses controlled shared memory with synchronization:

```
CPU0 ──► LMU (Local Memory Unit) ──► CPU1
              │
         (synchronization
          + integrity check)
```

---

## 9. ASIL-D Software Architecture

Hardware safety mechanisms alone are not sufficient. The software stack must also be
structured for safety compliance.

### Layered Software Architecture

```
┌──────────────────────────────────────┐
│          Application Layer           │  ◄── Motor control, BMS, etc.
├──────────────────────────────────────┤
│         Safety Middleware            │  ◄── OS, safety manager, error handler
├──────────────────────────────────────┤
│      MCAL (Microcontroller           │  ◄── AUTOSAR drivers (ADC, PWM, SPI...)
│      Abstraction Layer) Drivers      │
├──────────────────────────────────────┤
│     Hardware Safety Features         │  ◄── Lockstep, ECC, SMU, WDT, ENDINIT
└──────────────────────────────────────┘
```

### Required Software Safety Elements

| Element | Purpose |
|---|---|
| **Diagnostic routines** | Periodic self-tests of hardware (CPU test, RAM test) |
| **Error handling** | Classify, log, and react to all detected faults |
| **Start-up self-tests** | Validate hardware integrity before entering normal operation |
| **Monitoring tasks** | Periodic cross-checks of software execution flow |
| **Safe state handler** | Deterministic transition to a defined safe output state |

---

## 10. Automotive Example: EV Motor Controller

A concrete example of AURIX safety mechanisms in an electric vehicle motor controller:

### System Architecture

```
         ┌──────────────────────────────────────┐
         │               AURIX                  │
         │                                      │
Pedal ──►│  CPU0: Motor Control Algorithm       │──► PWM ──► Inverter ──► Motor
Sensor   │         │                            │
         │         ▼                            │
Rotor ──►│  CPU1: Lockstep (identical code)     │
Sensor   │         │                            │
         │         ▼                            │
         │  Comparator: CPU0 == CPU1 ?          │
         └──────────────────────────────────────┘
```

### Safety Mechanisms in Operation

```
1. LOCKSTEP CPU
   ─────────────
   CPU0 and CPU1 both compute torque demand.
   Comparator verifies match every cycle.
   Mismatch → SMU → motor disabled → safe stop.

2. ECC ON RAM
   ──────────
   Motor control tables and PID state stored in ECC RAM.
   Bit flip in PID integrator → auto-corrected (1-bit)
                              → SMU alarm (2-bit)

3. WATCHDOG
   ─────────
   Motor control ISR must kick watchdog every 1 ms.
   If ISR hangs → watchdog expires → CPU reset → motor stops.

4. SMU REACTION
   ─────────────
   Any alarm from any source → SMU evaluates configured reaction.
   For motor controller:
     - CPU lockstep error  → immediate reset + safe PWM off
     - ECC double-bit error → safe PWM off
     - Watchdog timeout    → reset
```

### Failure Scenario Walkthrough

```
Scenario: Cosmic ray causes bit flip in ALU output register of CPU0

Step 1:  CPU0 computes torque = 1200 Nm  (wrong, due to bit flip)
Step 2:  CPU1 computes torque = 800 Nm   (correct)
Step 3:  Comparator detects MISMATCH within same clock cycle
Step 4:  SMU alarm triggered immediately
Step 5:  Configured reaction: disable PWM outputs
Step 6:  Motor torque → 0 (safe state)
Step 7:  SMU triggers system reset
Step 8:  System restarts and re-enters normal operation (or limp home mode)

Time from fault to safe state: < 1 ms
```

---

## 11. Summary

AURIX achieves ASIL-D capability through **layered, independent, complementary**
safety mechanisms spanning hardware, firmware, and software.

### Safety Mechanism Summary

| Layer | Mechanism | What It Protects Against |
|---|---|---|
| **Processing** | Lockstep CPU | CPU computation errors, bit flips in ALU/registers |
| **Memory** | ECC (Flash, RAM, Cache) | Memory corruption, data integrity loss |
| **Software** | Watchdog (Safety + CPU) | Software hangs, infinite loops, missed deadlines |
| **System** | Safety Management Unit | Central fault collection and safe reaction |
| **Register** | ENDINIT Protection | Accidental or malicious modification of critical config |
| **Sensing** | Redundant Hardware | Sensor failures, stuck-at faults |

### Core Design Philosophy

```
Traditional approach:
    Assume the system will work correctly.

AURIX safety approach:
    Assume faults WILL occur.
    Detect them.
    Diagnose them.
    React safely.
    Enter a known safe state.
```

### The Safety Chain

```
Fault
  │
  ▼
Detect  (Lockstep / ECC / Watchdog / Monitor)
  │
  ▼
Diagnose  (SMU alarm classification)
  │
  ▼
React  (Reset / Interrupt / Safe Output)
  │
  ▼
Safe State  (Known, controlled, non-dangerous condition)
```

> AURIX does not aim to prevent all faults — hardware faults are physically unavoidable.
> It aims to ensure that **no single fault** can cause an undetected, uncontrolled dangerous
> condition. This is the core promise of ASIL-D.

---

*Based on ISO 26262 functional safety principles as applied to the Infineon AURIX TriCore
microcontroller family.*