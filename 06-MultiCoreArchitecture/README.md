# AURIX TriCore Multi-Core Architecture


## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Why AURIX Uses Multi-Core Architecture](#2-why-aurix-uses-multi-core-architecture)
- [3. TriCore Multi-Core Overview](#3-tricore-multi-core-overview)
- [4. AURIX Core Structure](#4-aurix-core-structure)
- [5. CPU Local Resources](#5-cpu-local-resources)
  - [5.1 Registers](#51-registers)
  - [5.2 CSA](#52-csa)
  - [5.3 PSPR](#53-pspr)
  - [5.4 DSPR](#54-dspr)
  - [5.5 Local Cache](#55-local-cache)
- [6. Shared System Resources](#6-shared-system-resources)
  - [6.1 LMU](#61-lmu)
  - [6.2 Flash Memory](#62-flash-memory)
  - [6.3 Peripherals](#63-peripherals)
- [7. System Resource Interconnect (SRI)](#7-system-resource-interconnect-sri)
- [8. Master Core and Slave Cores](#8-master-core-and-slave-cores)
- [9. Multi-Core Startup Sequence](#9-multi-core-startup-sequence)
- [10. Inter-Core Communication](#10-inter-core-communication)
- [11. Shared Memory Communication](#11-shared-memory-communication)
- [12. Synchronization Between Cores](#12-synchronization-between-cores)
- [13. Spinlocks](#13-spinlocks)
- [14. Interrupt-Based Communication](#14-interrupt-based-communication)
- [15. DMA Communication](#15-dma-communication)
- [16. Multi-Core Task Distribution Example](#16-multi-core-task-distribution-example)
- [17. Lockstep Architecture](#17-lockstep-architecture)
- [18. Automotive Example](#18-automotive-example)
- [19. Summary](#19-summary)


---

## 1. Introduction

Modern automotive systems require large processing capability.

Examples:

- Engine control
- Electric motor control
- ADAS systems
- Radar processing
- Communication gateways
- Diagnostics
- OTA updates


A single processor has limitations:

- Limited processing power
- Difficult real-time scheduling
- High interrupt load


AURIX solves this by integrating multiple TriCore processors inside one MCU.


Example:

```text
                 AURIX MCU


        +--------------------+

        |        CPU0        |

        |        CPU1        |

        |        CPU2        |

        |        CPU3        |

        +--------------------+

```


Each CPU can execute different software tasks independently.


---

# 2. Why AURIX Uses Multi-Core Architecture


## Performance Improvement


Instead of running everything on one CPU:


```text
CPU0

 |
 |
 +--> Motor Control

 +--> CAN

 +--> Diagnostics

 +--> Communication

```


Tasks are distributed:


```text
CPU0

 |
 +--> Motor Control



CPU1

 |
 +--> Communication



CPU2

 |
 +--> Diagnostics

```


This improves performance.


---

## Real-Time Determinism


Automotive systems require predictable timing.


Example:


```text
Motor Control ISR

Must execute every:

100 us

```


At the same time:


```text
CAN Messages

Ethernet

Diagnostics

```


are running.


Using multiple cores separates these workloads.


---

## Safety Isolation


Critical software can be separated.


Example:


```text
CPU0

Safety Control



CPU1

Communication

```


A failure in communication software is isolated from safety functions.


---

# 3. TriCore Multi-Core Overview


AURIX TC3xx devices contain:

- Multiple TriCore CPUs
- Private local memories
- Shared memories
- Shared peripherals
- Communication buses


High-level architecture:


```text

                         SRI


                          |

 ------------------------------------------------


       |                 |                 |


     CPU0              CPU1              CPU2


       |                 |                 |


     PSPR0             PSPR1             PSPR2


     DSPR0             DSPR1             DSPR2


    Cache0            Cache1            Cache2



                          |


                         LMU


```


---

# 4. AURIX Core Structure


Each TriCore CPU has private resources.


Example CPU0:


```text

             CPU0


 +------------------------+

 | Registers              |

 | CSA                    |

 | Instruction Cache      |

 | Data Cache             |

 | PSPR0                  |

 | DSPR0                  |

 +------------------------+


```


CPU1 and CPU2 have the same structure.


---

# 5. CPU Local Resources


Each core owns resources that are not directly shared.


## 5.1 Registers


Registers are the fastest storage.


Used for:


- Arithmetic operations
- Address calculations
- CPU state


---

## 5.2 CSA


CSA:

```
Context Save Area
```


Used for:


- Function calls
- Interrupt context saving
- Task switching


Each core has its own CSA area.


Example:


```text

CPU0

 |

CSA0


CPU1

 |

CSA1


```


---

## 5.3 PSPR


PSPR:

```
Program Scratch Pad RAM
```


Local program memory.


Used for:


- Critical ISRs
- Time-sensitive algorithms
- Safety functions


Example:


```text

CPU0


 |

 v


PSPR0


 |

Brake Control ISR


```


Advantages:


- Very fast
- No cache miss
- Deterministic execution


---

## 5.4 DSPR


DSPR:

```
Data Scratch Pad RAM
```


Local data memory.


Stores:


- Variables
- Stack
- Buffers
- RTOS objects


Example:


```text

CPU0


 |

 v


DSPR0


 |

Motor Speed Variable


```


Advantages:


- Low latency
- No bus contention
- Predictable timing


---

## 5.5 Local Cache


Each core has its own cache.


Example:


```text

CPU0 --> Cache0


CPU1 --> Cache1


CPU2 --> Cache2


```


Cache improves performance when accessing:


- Flash
- Shared memory


but it is not deterministic like PSPR/DSPR.


---

# 6. Shared System Resources


Resources accessible by all cores.


## 6.1 LMU


LMU:

```
Local Memory Unit
```


Shared RAM.


Example:


```text


CPU0
   \
    \
     LMU
    /
   /
CPU1


```


Used for:


- Inter-core communication
- Shared buffers
- Global data


---

## 6.2 Flash Memory


All cores can access:


```text
PFlash

DFlash

```


PFlash:

- Program code


DFlash:

- Calibration
- Persistent data


---

## 6.3 Peripherals


Examples:


- CAN
- SPI
- ADC
- UART
- Ethernet


Connected through the bus system.


---

# 7. System Resource Interconnect (SRI)


SRI:

```
System Resource Interconnect
```


It connects:


- CPUs
- Memory
- DMA
- Peripherals


Diagram:


```text


              SRI BUS


CPU0 --------|

CPU1 --------|

CPU2 --------|-------- Memory

DMA ---------|

Peripheral --|



```


SRI handles communication between system components.


---

# 8. Master Core and Slave Cores


After reset, one core starts first.


Usually:


```text
CPU0

Master Core

```


Responsibilities:


- Hardware initialization
- Clock setup
- Memory setup
- Start other cores


Other cores:


```text
CPU1

CPU2

CPU3

```


are started later.


---

# 9. Multi-Core Startup Sequence


Sequence:


```text

Reset


 |

 v


CPU0 Starts


 |

 v


Initialize System


 |

 v


Release CPU1


 |

 v


Release CPU2



```


CPU0 provides startup information to other CPUs.


---

# 10. Inter-Core Communication


Cores must exchange information.


Methods:


- Shared memory
- Interrupts
- Spinlocks
- DMA


Example:


```text

CPU0

Motor Control


CPU1

CAN Communication


```


They need to exchange data.


---

# 11. Shared Memory Communication


Example:


CPU0 writes:


```text

LMU


+----------------+

| Vehicle Speed  |

+----------------+


```


CPU1 reads:


```text

Vehicle Speed


```


Flow:


```text

CPU0

Write Data


 |

 v


LMU


 |

 v


CPU1

Read Data


```


---

# 12. Synchronization Between Cores


Problem:


Two CPUs access the same resource.


Example:


```text

CPU0:

counter++



CPU1:

counter++


```


Possible result:


```
Race Condition
```


Solutions:


- Spinlocks
- Mutex
- Atomic operations


---

# 13. Spinlocks


A spinlock allows one CPU to access a resource.


Example:


```text

CPU0


Acquire Lock


      |


Access Shared Resource


      |


Release Lock



CPU1


Wait



```


---

# 14. Interrupt-Based Communication


One core can notify another core.


Example:


```text

CPU0


Data Ready


 |

 v


Interrupt


 |

 v


CPU1


Process Data


```


Used for:


- Events
- Notifications
- Commands


---

# 15. DMA Communication


DMA can move data without CPU processing.


Example:


```text

CPU0


Data Buffer


 |

 v


DMA


 |

 v


CPU1 Memory


```


Benefits:


- Faster transfer
- Less CPU usage


---

# 16. Multi-Core Task Distribution Example


Example automotive ECU:


```text

              AURIX



CPU0

 |

 +--> Motor Control

 +--> Safety Functions



CPU1

 |

 +--> CAN

 +--> Ethernet



CPU2

 |

 +--> Diagnostics

 +--> Logging



```


Each core handles specific responsibilities.


---

# 17. Lockstep Architecture


Some AURIX devices support lockstep.


Two CPUs execute the same instructions.


Example:


```text


             Program


                |


        ----------------


        |              |


       CPU0           CPU1


     Execute        Execute


        |              |


        ---- Compare ---



```


If results differ:


```text

Fault Detected


```


Used for:


- Safety systems
- ASIL applications


---

# 18. Automotive Example


Vehicle controller:


```text

                  AURIX


CPU0

 |

Motor Control


CPU1

 |

Communication Gateway


CPU2

 |

Diagnostics


      


LMU

 |

Shared Vehicle Data



```


---

# 19. Summary


AURIX multi-core architecture provides:


## Private Resources


```text

Registers

CSA

Cache

PSPR

DSPR

```


## Shared Resources


```text

LMU

Flash

Peripherals

SRI

```


## Communication Methods


```text

Shared Memory

Interrupts

Spinlocks

DMA

```


## Safety Features


```text

Lockstep

ECC

Fault Detection

```


The main idea:


```text

Multiple Cores

+

Fast Local Memory

+

Shared Communication

+

Safety Mechanisms


=

Real-Time Automotive Computing Platform


```