SYSTEM ARCHITECTURE & SCOPING DOCUMENT
Project Chameleon RTOS
A Modular, Hot-Swappable Real-Time Operating System Kernel for the STM32F103
Architecture

Target Hardware: STM32F103RBT6 (ARM Cortex-M3) Author: Lead Systems Architect / Embedded OS Developer
Memory Profiles: 128 KB Flash, 20 KB SRAM Classification: Technical Research & Blueprint

1. Executive Summary & Vision
The objective of Project Chameleon is to break away from standard static embedded architectures by

engineering a Real-Time Operating System (RTOS) capable of performing dynamic, over-the-wire or over-the-
air runtime module updating without system reboots. Operating within the tight constraints of the ARM Cortex-
M3 microcontroller (specifically the STM32F103RB), Chameleon transforms an asset traditionally treated as

fixed firmware into an adaptive application platform.
By treating the kernel as a permanent host infrastructure and applications as decoupled, hot-swappable
binary payloads, this architecture eliminates the downtime, risk, and bandwidth overhead of monolithic
firmware flashing.

2. Core Problem Statement

The Monolithic Firmware Trap: Traditional microcontroller development compiles the OS, drivers,
stacks, and application logic into a single, unified .bin or .hex file. When an upgrade to a single minor
application feature is required, the entire controller must be taken offline, erased, reprogrammed, and
rebooted.

Impact of Existing Paradigms:
System Downtime: During flash erasing and writing cycles (which take several seconds), real-time control
loops are severed, presenting high operational risks in industrial or medical equipment.
Flash Wear-out: Rewriting a whole 128 KB block to change a 2 KB routine drastically reduces the lifetime
of the internal flash peripheral.
Bandwidth Inefficiency: Pushing a full 100 KB binary update over slow industrial fieldbuses (RS-485/
CAN) or wireless networks (LoRaWAN/BLE) incurs severe communication overhead.
•

•

•

Chameleon RTOS Architecture Exploration Document Page 1 of 5

3. The Solution: Dynamic Application Isolation

The Chameleon Blueprint: Segmenting the microcontroller’s non-volatile storage into an immutable,
highly robust "Kernel Space" and a transient, page-aligned "Dynamic Application Window". Application
modules are compiled as standalone, position-independent or fixed-offset binaries and hot-swapped live
via runtime linking tables.

Key Value Propositions:
Zero-Reboot Lifecycle: The scheduler never pauses. Non-updated tasks continue meeting real-time
deadlines while a target app slot is hot-swapped.
Granular Flashing: Flash write operations are isolated strictly to designated pages, protecting the primary
operating system from corruption.
Ultra-Low Bandwidth Footprint: Updates shrink from tens or hundreds of kilobytes down to highly
optimized 2-4 KB binary payloads containing exclusively the target application code.

4. Hardware Constraints Analysis (STM32F103RBT6)

Designing a dynamic loader requires intimate alignment with the physical memory layout of the ARM Cortex-
M3 core on our target silicon:

Resource
Component

Physical Limitation Architectural Allocation strategy

On-Chip Flash
Memory

128 KB (Divided into 128 pages of
1 KB each)

Pages 0 to 107 (108 KB): Kernel and Static
Assets.
Pages 108 to 127 (20 KB): Dynamic App Partition.

Internal SRAM 20 KB total size

16 KB: Kernel data structures, system stacks,
heap.
4 KB: Allocated dynamic app stack and localized
data.

Flash Programming
Interface

Minimum 16-bit Half-Word writes;
page erase requirement

Managed via internal Flash Control Register
(FLASH_CR) with lock/unlock sequences.

5. Memory Topology & Partitioning
The physical layout must guarantee that an interruption or corruption in the app segment cannot interfere with
the kernel's boot execution paths.
1.

2.

3.

Chameleon RTOS Architecture Exploration Document Page 2 of 5

Vector Table
(0x0800_0000)

Immutable Kernel & RTOS Engine
(Schedulers, HAL, Drivers, Tasks)
Size: 106 KB

Dynamic App
Window
(Pages 108-127)
Base:
0x0801_B000

Opt
Flash
(1
KB)

Figure 1: Proposed Physical Segmentation of the STM32F103 Internal Flash Memory Map

6. Architectural Mechanisms
A. The Shared System Vector Table (SVTs)
Because the Dynamic Application is compiled independently of the kernel, it cannot directly call kernel
functions like os_delay() or gpio_write() using absolute addresses. To resolve this, the kernel exposes
a fixed Jump Table at a known address (e.g., the very start of the Application window or a reserved sector in
RAM).

typedef struct {
void (*task_yield)(void);
void (*log_printf)(const char *fmt, ...);
int (*gpio_toggle)(uint16_t pin);
void *(*malloc)(size_t size);
} KernelSVT_t;
// Located at a deterministic, immutable hardware boundary
#define KERNEL_SVT_ADDRESS ((KernelSVT_t*)0x0801AC00)

B. Decoupling Compilation Options
To make the binary executable anywhere or within its window without link-time resolution, we can deploy two
main compile approaches:
Position-Independent Code (-fPIC): Generates address-agnostic code using a Global Offset Table
(GOT). The runtime loader recalculates addresses on the fly based on where the app is loaded into
memory.
Fixed-Base Application Target: A much simpler and lower-overhead approach for our chip. The compiler
forces the dynamic application to link strictly to a fixed start address (0x0801B000). The app will *always*
execute from this page boundaries, entirely bypassing complex runtime relocation overhead.
•

•

Chameleon RTOS Architecture Exploration Document Page 3 of 5

7. The Runtime Dynamic Loading Lifecycle
The hot-swap operation is governed by a state machine run by a low-priority system worker thread within the
kernel:
Ingress Phase: The new application binary payload is streamed over a peripheral interface (e.g., UART at
115200 bps) in chunks. Each chunk is accompanied by a CRC-32 checksum.
Staging & Verification: The payload is temporarily cached either in an isolated SRAM block or verified
stream-by-stream. The kernel parses a custom 8-byte Chameleon Header appended to the front of the
binary to verify magic numbers and expected target offsets.
Suspension: The scheduler temporarily halts execution of the *previous* version of the dynamic task and
clears its context block from the run queue.
Flash Mutex & Write:
The kernel unlocks the flash peripheral registers via FLASH_KEY1 and 2.
An erase command is executed on pages 108 through 127.
The new code blocks are written sequentially into Flash.
Registration & Wake: The kernel re-initializes the dynamic task control block (TCB), points the Task Entry
Pointer to 0x0801B008 (skipping the header), clears its stack frame, and marks the task as active. The
scheduler transparently resumes execution.

8. Execution Protection & Mitigations
Executing un-linked code compiled outside the main pipeline carries high operational risk. Chameleon
implements protective guards to preserve system continuity:
The HardFault Safety Net: If the application tries to execute an illegal opcode, branch to unmapped
space, or write outside allowed memory, the Cortex-M3 throws a hardware exception. Chameleon
overloads the default HardFault_Handler to safely terminate the dynamic app slot, log the fault reason,
and restart the system core cleanly.
Watchdog Integration: An Independent Watchdog (IWDG) timer runs continuously. If a newly flashed
dynamic application enters an infinite execution loop, the kernel will fail to clear the IWDG, forcing a
hardware reset back into a clean, safe kernel-only baseline configuration.

9. Next Steps & Action Plan
To execute this project step-by-step, we will adopt an incremental build strategy:
Phase 1: Configure a basic preemptive RTOS kernel on the STM32F103 utilizing PendSV and SysTick
handlers for context switching.
Phase 2: Implement internal Flash read/write drivers and create a firm layout boundary using custom linker
scripts (linker_script.ld).
1.

2.

3.

4.
◦
◦
◦
5.

•

•

1.

2.

Chameleon RTOS Architecture Exploration Document Page 4 of 5

Phase 3: Build the host-side Python tool to package compiled application binaries into Chameleon-
compliant images.

Phase 4: Write
