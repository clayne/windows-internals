# Chapter 3 System Mechanisms

## Trap Dispatching

- __Interrupts__ and __exceptions__ are OS conditions that divert the processor to code outside the normal flow of control.
- The term __trap__ refers to a processor’s mechanism for capturing an executing thread when an exception or an interrupt occurs and transferring control to a fixed location in the OS.
- The processor transfers control to a __trap handler__, which is a function specific to a particular interrupt or exception.
- The kernel distinguishes between interrupts and exceptions in the following way:
    - An interrupt is an __asynchronous__ event (one that can occur at any time) that is unrelated to what the processor is executing.
        - Generated primarily by __I/O devices, processor clocks, or timers__, and they can be enabled or disabled.
    - An exception, in contrast, is a __synchronous__ condition that usually results from the execution of a particular instruction.
        - Running a program a second time with the same data under the same conditions can reproduce exceptions.
        - Examples of exceptions include __memory-access violations__, __certain debugger instructions__, and __divideby-zero__ errors.
        - The kernel also regards system service calls as exceptions (although technically they’re system traps).
    <p align="center"><img src="./assets/trap-handlers.png" width="400px" height="auto"></p>
- Either hardware or software can generate exceptions and interrupts:
    - A __bus error__ exception is caused by a hardware problem, whereas a __divide-by-zero__ exception is the result of a software bug.
    - An __I/O device__ can generate an interrupt, or the kernel itself can issue a software interrupt such as an __APC or DPC__.
- When a hardware exception or interrupt is generated, the processor records enough machine state on the __kernel stack of the thread__ that’s interrupted to return to that point in the control flow and continue execution as if nothing had happened. If the thread was executing in user mode, Windows __switches to the thread’s kernel-mode stack__.
- Windows then creates a __trap frame__ on the kernel stack of the interrupted thread into which it stores the execution state of the thread
    ```c
    0: kd> dt nt!_ktrap_frame
    +0x000 P1Home           : Uint8B
    +0x008 P2Home           : Uint8B
    +0x010 P3Home           : Uint8B
    +0x018 P4Home           : Uint8B
    +0x020 P5               : Uint8B
    +0x028 PreviousMode     : Char
    +0x029 PreviousIrql     : UChar
    +0x02a FaultIndicator   : UChar
    +0x02a NmiMsrIbrs       : UChar
    +0x02b ExceptionActive  : UChar
    +0x02c MxCsr            : Uint4B
    +0x030 Rax              : Uint8B
    ...
    ```

### Interrupt Dispatching

- Hardware-generated interrupts typically originate from I/O devices that must notify the processor when they need service.
    - Interrupt-driven devices allow the OS to get the __maximum use out of the processor__ by overlapping central processing with I/O operations.
    - A thread starts an I/O transfer to or from a device and then can execute other useful work while the device completes the transfer.
    - When the device is finished, it interrupts the processor for service.
    - Pointing devices, printers, keyboards, disk drives, and network cards are generally __interrupt driven__.
- The kernel installs interrupt trap handlers to respond to device interrupts.
    - Those handlers transfer control either to an external routine (__Interrupt Service Routine ISR__) that handles the interrupt or ;
    - To an internal kernel routine that responds to the interrupt.
    - Device drivers supply ISRs to service device interrupts, and the kernel provides interrupt-handling routines for other types of interrupts.

#### Hardware Interrupt Processing

- On the hardware platforms supported by Windows, external I/O interrupts come into one of the lines on an __interrupt controller__.
- The controller, in turn, interrupts the processor on a single line.
- Once the processor is interrupted, it queries the controller to get the __interrupt request (IRQ)__.
- The interrupt controller translates the IRQ to an interrupt number, uses this number as an index into a structure called the __interrupt dispatch table (IDT)__, and transfers control to the appropriate interrupt dispatch routine.
- At system boot time, Windows fills in the IDT with pointers to the kernel routines that handle each interrupt and exception.
- Windows maps hardware IRQs to interrupt numbers in the IDT, and the system also uses the IDT to configure trap handlers for exceptions.
```c
kd> !idt

Dumping IDT: fffff8000a0d1000

00:	fffff80002be9100 nt!KiDivideErrorFaultShadow
01:	fffff80002be9180 nt!KiDebugTrapOrFaultShadow	Stack = 0xFFFFF8000A0D49E0
02:	fffff80002be9200 nt!KiNmiInterruptShadow	Stack = 0xFFFFF8000A0D47E0
03:	fffff80002be9280 nt!KiBreakpointTrapShadow
04:	fffff80002be9300 nt!KiOverflowTrapShadow
05:	fffff80002be9380 nt!KiBoundFaultShadow
06:	fffff80002be9400 nt!KiInvalidOpcodeFaultShadow
07:	fffff80002be9480 nt!KiNpxNotAvailableFaultShadow
08:	fffff80002be9500 nt!KiDoubleFaultAbortShadow	Stack = 0xFFFFF8000A0D43E0
09:	fffff80002be9580 nt!KiNpxSegmentOverrunAbortShadow
0a:	fffff80002be9600 nt!KiInvalidTssFaultShadow
0b:	fffff80002be9680 nt!KiSegmentNotPresentFaultShadow
0c:	fffff80002be9700 nt!KiStackFaultShadow
0d:	fffff80002be9780 nt!KiGeneralProtectionFaultShadow
0e:	fffff80002be9800 nt!KiPageFaultShadow
...
```
- Each processor has a separate IDT so that different processors can run different ISRs, if appropriate.

#### x86 Interrupt Controllers

- Most x86 systems rely on either the __i8259A Programmable Interrupt Controller (PIC)__ or a variant of the __i82489 Advanced Programmable Interrupt Controller (APIC)__.
- Today’s computers include an APIC.
- PIC:
    - Works only on __uni-processors__ systems.
    - Has only __eight__ interrupt lines
    - IBM PC arch extended it to __15__ interrupt lines (7 on master + 8 on slave).
- APIC:
    - Work with __multiprocessor__ systems.
    - Have __256__ interrupt lines. I
    - For compatibility, APIC supports a PIC mode.
    - Consists of several components:
        - __I/O APIC__ that receives interrupts from devices
        - __Local APICs__ that receive interrupts from the I/O APIC on the bus and that interrupt the CPU they are associated with.
        - An i8259A-compatible interrupt controller that translates APIC input into PIC-equivalent signals.
        <p align="center"><img src="./assets/apic.png" width="300px" height="auto"></p>
- x64 Windows will not run on systems that do not have an APIC because they use the APIC for interrupt control.
- IA64 architecture relies on the __Streamlined Advanced Programmable Interrupt Controller (SAPIC)__, which is an evolution of the APIC. Even if __load balancing__ and __routing__ are present in the firmware, Windows does not take advantage of it; instead, it __statically__ assigns interrupts to processors in a __round-robin__ manner.
```c
0: kd> !pic
----- IRQ Number ----- 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
Physically in service:  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .
Physically masked:      Y  Y  Y  Y  Y  Y  Y  Y  Y  Y  Y  Y  Y  Y  Y  Y
Physically requested:   Y  .  .  .  .  .  .  .  .  .  .  .  Y  .  .  .
Level Triggered:        .  .  .  .  .  .  .  Y  .  Y  Y  Y  .  .  .  .


0: kd> !apic
Apic @ fffe0000  ID:0 (60015)  LogDesc:01000000  DestFmt:ffffffff  TPR F0
TimeCnt: 00000000clk  SpurVec:3f  FaultVec:e3  error:0
Ipi Cmd: 02000000`00000c00  Vec:00  NMI       Lg:02000000      edg high
Timer..: 00000000`000300fd  Vec:FD  FixedDel    Dest=Self      edg high      m
Linti0.: 00000000`0001003f  Vec:3F  FixedDel    Dest=Self      edg high      m
Linti1.: 00000000`000004ff  Vec:FF  NMI  Dest=Self      edg high
TMR: 66, 76, 86, 96, B1
IRR: 2F, 96, D1
ISR: D1

0: kd> !ioapic
IoApic @ FEC00000  ID:2 (20)  Arb:2000000
Inti00.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti01.: 03000000`00000981  Vec:81  LowestDl  Lg:03000000      edg high
Inti02.: 01000000`000008d1  Vec:D1  FixedDel  Lg:01000000      edg high
Inti03.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti04.: 03000000`00000961  Vec:61  LowestDl  Lg:03000000      edg high
Inti05.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti06.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti07.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti08.: 01000000`000009d2  Vec:D2  LowestDl  Lg:01000000      edg high
Inti09.: 03000000`0000a9b1  Vec:B1  LowestDl  Lg:03000000      lvl low
Inti0A.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti0B.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti0C.: 03000000`00000971  Vec:71  LowestDl  Lg:03000000      edg high
Inti0D.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti0E.: 03000000`00000975  Vec:75  LowestDl  Lg:03000000      edg high
Inti0F.: 03000000`00000965  Vec:65  LowestDl  Lg:03000000      edg high
Inti10.: 03000000`0000a976  Vec:76  LowestDl  Lg:03000000      lvl low
Inti11.: 03000000`0000a986  Vec:86  LowestDl  Lg:03000000      lvl low
Inti12.: 03000000`0000e966  Vec:66  LowestDl  Lg:03000000      lvl low  rirr
Inti13.: 03000000`0000e996  Vec:96  LowestDl  Lg:03000000      lvl low  rirr
Inti14.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti15.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti16.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
Inti17.: 00000000`000100ff  Vec:FF  FixedDel  Ph:00000000      edg high      m
```

#### Software Interrupt Request Levels (IRQLs)

- Although interrupt controllers perform interrupt prioritization, Windows __imposes__ its own interrupt __priority__ scheme known as __interrupt request levels (IRQLs)__.
- The kernel represents IRQLs internally as a number from 0 through 31 on x86 and from 0 to 15 on x64 and IA64, with __higher numbers__ representing __higher-priority__ interrupts. <p align="center"><img src="./assets/irqls.png" width="600px" height="auto"></p>
- Interrupts are serviced in priority order, and a higher-priority interrupt preempts the servicing of a lower-priority interrupt.
- When a high-priority interrupt occurs, the processor saves the __interrupted thread’s state__ and invokes the trap dispatchers associated with the interrupt.
- The trap dispatcher raises the IRQL and calls the interrupt’s service routine.
- After the service routine executes, the interrupt dispatcher lowers the processor’s IRQL to where it was before the interrupt occurred and then loads the saved machine state. The interrupted thread resumes executing where it left off.
- When the kernel lowers the IRQL, lower-priority interrupts that were masked might materialize. If this happens, the kernel repeats the process to handle the new interrupts.
- IRQLs are also used to __synchronize__ access to kernel-mode data structures. As a kernel-mode thread runs, it raises or lowers the processor’s IRQL either directly by calling `KeRaiseIrql` and `KeLowerIrql` or, more commonly, indirectly via calls to functions that acquire kernel synchronization objects.
- For example, when an interrupt occurs, the trap handler (or perhaps the processor) raises the processor’s IRQL to the assigned IRQL of the interrupt source. This elevation __masks__ all interrupts __at and below that IRQL__ (on that processor only), which ensures that the processor servicing the interrupt isn’t waylaid by an interrupt at the __same level or a lower level__. The masked interrupts are either handled by another processor or held back until the IRQL drops. Therefore, all components of the system, including the kernel and device drivers, attempt to __keep the IRQL at passive level__ (sometimes called low level). They do this because device drivers can respond to hardware interrupts in a timelier manner if the IRQL isn’t kept __unnecessarily elevated for long periods__.
-  > :bangbang: An exception to the rule that raising the IRQL blocks interrupts of that level and lower relates to APC-level interrupts If a thread raises the IRQL to APC level and then is rescheduled because of a dispatch/DPC-level interrupt, the system might deliver an APC-level interrupt to the newly scheduled thread Thus, APC level can be considered a thread-local rather than processor-wide IRQL.
-  Processor’s IRQL is always at __passive level__ when it’s executing usermode code. Only when the processor is executing kernel-mode code can the IRQL be higher.

#### Mapping Interrupts to IRQLs

- IRQL levels aren’t the same as the interrupt requests (IRQs) defined by interrupt controllers.
- In HAL in Windows, a type of device driver called a __bus driver__ determines the presence of devices on its bus (PCI, USB, and so on) and what interrupts can be assigned to a device.
- The bus driver reports this information to the __Plug and Play__ manager, which decides, after taking into account the acceptable interrupt assignments for all other devices, which interrupt will be assigned to each device.
- Then it calls a Plug and Play interrupt arbiter, which __maps interrupts to IRQLs__.

#### Predefined IRQLs

- The kernel uses __high level__ only when it’s halting the system in `KeBugCheckEx` and masking out all interrupts.
- __Power fail level__ originated in the original Windows NT design documents, which specified the behavior of system power failure code, but this IRQL has never been used.
- __Interprocessor interrupt level__ is used to request another processor to perform an action, such as updating the processor’s TLB cache, system shutdown, or system crash.
- __Clock level__ is used for the system’s clock, which the kernel uses to track the time of day as well as to measure and allot CPU time to threads.
- The system’s real-time clock (or another source, such as the local APIC timer) uses __profile level__ when kernel profiling (a performance-measurement mechanism) is enabled.
- The __synchronization IRQL__ is internally used by the dispatcher and scheduler code to protect access to global thread scheduling and wait/synchronization code.
- The __device IRQLs__ are used to prioritize device interrupts.
- The __corrected machine check interrupt level__ is used to signal the OS after a serious but corrected hardware condition or error that was reported by the CPU or firmware through the Machine Check Error (MCE) interface.
- __DPC/dispatch-level and APC-level interrupts__ are software interrupts that the kernel and device drivers generate.
- The lowest IRQL, __passive level__, isn’t really an interrupt level at all; it’s the setting at which normal thread execution takes place and all interrupts are allowed to occur.
> ‼️ One important restriction on code running at DPC/dispatch level or above is that it can’t wait for an object if doing so necessitates the scheduler to select another thread to execute, which is an illegal operation because the scheduler relies on DPC-level software interrupts to schedule threads.

> ‼️ Another restriction is that only nonpaged memory can be accessed at IRQL DPC/dispatch level or higher. This rule is actually a side effect of the first restriction because attempting to access memory that isn’t resident results in a page fault. When a page fault occurs, the memory manager initiates a disk I/O and then needs to wait for the file system driver to read the page in from disk. This wait would, in turn, require the scheduler to perform a context switch (perhaps to the idle thread if no user thread is waiting to run), thus violating the rule that the scheduler can’t be invoked (because the IRQL is still DPC/dispatch level or higher at the time of the disk read). A further problem results in the fact that I/O completion typically occurs at APC_LEVEL, so even in cases where a wait wouldn’t be required, the I/O would never complete because the completion APC would not get a chance to run.

#### Interrupt Objects

- The kernel provides a **portable** mechanism —a kernel control object called an **interrupt object**— that allows device drivers to **register ISRs** for their devices.
- An interrupt object contains all the information the kernel needs to associate a device ISR with a particular level of interrupt, including the **address of the ISR**, the **IRQL** at which the **device interrupts**, and the **entry** in the kernel’s interrupt dispatch table (**IDT**) with which the ISR should be associated.
- When an interrupt object is initialized, a few instructions of assembly language code, called the **dispatch code**, are copied from an interrupt-handling template, `KiInterruptTemplate`, and stored in the object . When an interrupt occurs, this code is executed.
<p align="center"><img src="./assets/interupt-control-flow.png" width="300px" height="auto"></p>

To view the contents of the interrupt object associated with the interrupt, execute dt `nt!_kinterrupt` with the address following `KINTERRUPT` (get it via `!idt`):

```c
lkd> dt nt!_KINTERRUPT fffffa80045bad80
	+0x000 Type             : 22
	+0x002 Size             : 160
    +0x008 InterruptListEntry : _LIST_ENTRY [ 0x0000000000000000 - 0x0 ]
    +0x018 ServiceRoutine   : 0xfffff8800356ca04     unsigned char i8042prt!I8042KeyboardInterruptService+0 // <-- The ISR’s address
    +0x020 MessageServiceRoutine : (null)
	+0x028 MessageIndex
	+0x030 ServiceContext
	+0x038 SpinLock
	+0x040 TickCount
	+0x048 ActualLock
	+0x050 DispatchAddress  : 0xfffff80001a7db90 //
	+0x058 Vector           : 0x81
	+0x05c Irql             : 0x8 ''
	...
	+0x090 DispatchCode		: [4] 0x8d485550 // <-- interrupt code that executes when an interrupt occurs,
											 // it build the trap frame on the stack and then call the function
											 // stored in the DispatchAddress passing it a pointer to the interrupt object .
```

- Although there is **no direct mapping** between an **interrupt vector** and an **IRQ**, Windows does keep track of this translation when managing device resources through what are called *arbiters*.
- For each resource type, an arbiter maintains the relationship between **virtual resource** usage (such as an interrupt vector) and **physical resources** (such as an interrupt line).
- You can query the ACPI IRQ arbiter with the `!apciirqarb` command to obtain information on the ACPI IRQ arbiter.

```c
lkd> !acpiirqarb
	Processor 0 (0, 0):
	Device Object: 0000000000000000
	Current IDT Allocation:
	...
	0000000000000081 - 0000000000000081 D fffffa80029b4c20 (i8042prt) A:0000000000000000 IRQ:0
...
```

- You will be given the owner of the vector, in the type of a device object. You can then use the `!devobj` command to get information on the `i8042prt` device in this example (which corresponds to the PS/2 driver):

```c
lkd> !devobj fffffa80029b4c20

...
Entry 4 - Interrupt (0x2) Device Exclusive (0x1)
      Flags (0x01) - LATCHED
      Level 0x1, Vector 0x1, Group 0, Affinity 0xffffffff
```

> ‼️ Windows and Real-Time Processing
> Because Windows doesn’t enable controlled prioritization of device IRQs and user-level applications execute only when a processor’s IRQL is at passive level, Windows **isn’t** typically suitable as a **real-time OS**.
- Associating or disconnecting an ISR with a particular level of interrupt is called connecting or disconnecting an interrupt object.
	- Accomplished by calling `IoConnectInterruptEx` and `IoDisconnectInterruptEx`, which allow a device driver to “turn on” an ISR when the driver is loaded into the system and to “turn off” the ISR if the driver is unloaded.
	- :+1: Prevents device drivers from fiddling directly with interrupt hardware.
	- :+1: Aids in creating portable device drivers.

- Interrupt objects allow the kernel to easily call more than one ISR for any interrupt level.
	- Multiple device drivers create interrupt objects and connect them to the same IDT entry.
	- Allows the kernel to support *daisy-chain* configurations.

<details><summary>Line-Based vs. Message Signaled-Based Interrupts:</summary>

- Shared interrupts are often the cause of high interrupt latency and can also cause stability issues.
- Consuming four IRQ lines for a single device quickly leads to **IRQ line exhaustion**.
- PCI devices are each connected to only one IRQ line anyway, so we cannot use more than one IRQ in the first place.
- Provide poor scalability in multiprocessor environments.
- Message-signaled interrupts (MSI) model solves these issue: a device delivers a message to its driver by **writing to a specific memory address**.
- This action causes an interrupt, and Windows then calls the ISR with the message content (value) and the address where the message was delivered.
- A device can also deliver multiple messages (up to 32) to the memory address, delivering different payloads based on the event.
- The need for IRQ lines is removed ▶️ decreasing latency.
- Nullifies any benefit of sharing interrupts, decreasing latency further by directly delivering the interrupt data to the concerned ISR.
- MSI-X, an extension to the MSI model, introduced in PCI v3.0
- Add the ability to use a different address (which can be dynamically determined) for each of the MSI payloads ▶️ enabling NUMA.
- Improves latency and scalability.
</details>

<details><summary>Interrupt Affinity and Priority:</summary>

- On systems that both support ACPI and contain an APIC, Windows enables driver developers and administrators to somewhat control the processor affinity (selecting the processor or group of processors that receives the interrupt) and affinity policy (selecting how processors will be chosen and which processors in a group will be chosen).
- Configurable through a registry value called `InterruptPolicyValue` in the Interrupt Management\Affinity Policy key under the device’s instance key in the registry.
</details>

#### Software Interrupts

- Although hardware generates most interrupts, the Windows kernel also generates **software interrupts** for a variety of tasks:
	- Initiating thread dispatching
	- Non-time-critical interrupt processing
	- Handling timer expiration
	- Asynchronously executing a procedure in the context of a particular thread
	- Supporting asynchronous I/O operations

##### Dispatch or Deferred Procedure Call (DPC) Interrupts

- The kernel always raises the processor’s IRQL to DPC/dispatch level or above when it needs to **synchronize access to shared** kernel structures.
- When the kernel detects that **dispatching should occur**, it requests a DPC/dispatch-level interrupt.
-  The kernel uses DPCs to process **timer expiration** (and release threads waiting for the timers) and to **reschedule the processor** after a thread’s quantum expires.
- Device drivers use DPCs to process interrupts:
	- Perform the minimal work necessary to acknowledge their device, save volatile interrupt state, and defer data transfer or other less time-critical interrupt processing activity for execution in a DPC at DPC/dispatch IRQL.
- By default, the kernel places DPC objects at the end of the DPC queue of the **processor on which the DPC was requested** (typically the processor on which the **ISR executed**).
<p align="center"><img src="./assets/delivering-a-dpc.png" width="500px" height="auto"></p>

- DPC routines execute without regard to what thread is running, meaning that when a DPC routine runs, it **can’t** assume what process address space is **currently mapped**.
- DPC routines can call kernel functions, but they **can’t call system services**, generate page faults, or create or wait for dispatcher objects. They can, however, access nonpaged system memory addresses, because system address space is always mapped regardless of what the current process is.
DPCs are provided **primarily for device drivers**, but the kernel uses them too . The kernel most frequently uses a DPC to **handle quantum expiration**.
- At every tick of the system clock, an interrupt occurs at clock IRQL. The *clock interrupt handler* updates the system time
and then decrements a counter that tracks how long the current thread has run. When the counter **reaches 0**, the thread’s time quantum has expired and the kernel might need to reschedule the processor, a lower-priority task that should be done at DPC/dispatch IRQL.
- ‼️ Because DPCs execute **regardless of whichever thread is currently running** on the system, they are a primary cause for perceived system **unresponsiveness** of client systems or workstation workloads because even the highest-priority thread will be interrupted by a pending DPC. Some DPCs run long enough that users might perceive **video or sound lagging**, and even **abnormal mouse or keyboard latencies**, so for the benefit of drivers with long-running DPCs, Windows supports **threaded DPCs**.
- Threaded DPCs function by executing the DPC routine at **passive level** on a **real-time priority (priority 31) thread**.

##### Asynchronous Procedure Call (APC) Interrupts

- APCs provide a way for user programs and system code **to execute in the context of a particular user thread** (and hence a particular process address space).
- Because APCs are queued to execute in the context of a particular thread and run at an IRQL less than DPC/dispatch level, they don’t operate under the same restrictions as a DPC.
	- An APC routine can **acquire** resources (objects), **wait** for object handles, incur **page faults**, and call **system services**.
- Unlike the DPC queue, which is **systemwide**, the APC queue is **thread-specific** — each thread has its own APC queue.
- There are two kinds of APCs: **kernel mode** and **user mode**:
- Kernel mode APCs:
	- They don’t require **permission** from a target thread to run in that thread’s context, while user-mode APCs do.
	- Two types of kernel-mode APCs: **normal** and **special** .
		- Special APCs execute at **APC level** and allow the APC routine to **modify** some of the APC **parameters**.
		- Normal APCs execute at **passive level** and receive the modified parameters from the special APC routine (or the original parameters if they weren’t modified).
- Both normal and special APCs can be **disabled** by **raising the IRQL** to **APC level** or by calling `KeEnterGuardedRegion`.
- `KeEnterGuardedRegion` disables APC delivery by setting the `SpecialApcDisable` field in the calling thread’s `KTHREAD` structure.
- A thread can **disable normal APCs only** by calling `KeEnterCriticalRegion`, which sets the `KernelApcDisable` field in the thread’s `KTHREAD` structure.
- Kernel-mode APCs uses cases by the executive:
	- Direct a thread to stop executing an interruptible system service.
	- Record the results of an asynchronous I/O operation in a thread’s address space.
- Environment subsystems use special kernel-mode APCs to make a thread suspend or terminate itself or to get or set its user-mode execution context.
- Several Windows APIs—such as `ReadFileEx`, `WriteFileEx`, and `QueueUserAPC` use **user-mode APCs**.
	- For example, the `ReadFileEx` and `WriteFileEx` allow the caller to specify a **completion routine** to be called when the I/O operation finishes.
	- The I/O completion is implemented by queuing an APC to the thread that issued the I/O.
	- However, the callback to the completion routine doesn’t necessarily take place when the APC is queued because user-mode APCs are delivered to a thread only when it’s in an **alertable wait state**.
	- A thread can enter a wait state either by **waiting for an object handle** and specifying that its **wait is alertable** (with the Windows `WaitForMultipleObjectsEx()`) or by testing directly whether it has a pending APC (using `SleepEx()`).
	- In both cases, if a user-mode APC is pending, the kernel interrupts (alerts) the thread, transfers control to the APC routine, and resumes the thread’s execution when the APC routine completes.

### Timer Processing

- The system’s clock interval timer is probably the **most important device** on a Windows machine, as evidenced by its high IRQL value (`CLOCK_LEVEL`) and due to the critical nature of the work it is responsible for. Without this interrupt:
	- Windows would lose track of time, causing erroneous results in calculations of **uptime** and **clock time**;
	- And worst, causing timers **not to expire** anymore and **threads** never to **lose** their **quantum** anymore.
	- :arrow_forward: Windows would also not be a preemptive OS, and unless the current running thread yielded the CPU, critical background tasks and scheduling could never occur on a given processor.
- Windows programs the system clock to fire at the **most appropriate interval** for the machine, and subsequently allows drivers, applications, and administrators to **modify the clock interval** for their needs.
- The system clock is maintained either by the *PIT (Programmable Interrupt Timer)* chip that is present on all computers since the PC/AT, or the *RTC (Real Time Clock)*.
- On today’s machines, the APIC Multiprocessor HAL configures the RTC to fire every 15.6 milliseconds, which corresponds to about 64 times a second.

<details><summary>Identifying High-Frequency Timers:</summary>

- Windows uses ETW to trace all processes and drivers that request a change in the system’s clock interval, displaying the time of the occurrence and the requested interval.
- Can help identifying the causes of poor battery performance on otherwise healthy system.
- To obtain it, simply run `powercfg /energy` ; or with a debugger, for each process, the `EPROCESS` structure contains a number of fields that help identify changes in timer resolution:
```c
   +0x4a8 TimerResolutionLink : _LIST_ENTRY [ 0xfffffa80'05218fd8 - 0xfffffa80'059cd508 ]
   +0x4b8 RequestedTimerResolution : 0
   +0x4bc ActiveThreadsHighWatermark : 0x1d
   +0x4c0 SmallestTimerResolution : 0x2710
   +0x4c8 TimerResolutionStackRecord : 0xfffff8a0'0476ecd0 _PO_DIAG_STACK_RECORD
   // Then:
   lkd> !list "-e -x \"dt nt!_EPROCESS @$extret-@@(#FIELD_OFFSET(nt!_EPROCESS,TimerResolutionLink))
```
</details>

#### Timer Expiration

- One of the main tasks of the ISR associated with the interrupt that the RTC or PIT will generate is to **keep track of system time**, which is mainly done by the `KeUpdateSystemTime` routine.
- Its second job is to keep track of **logical run time**, such as **process/thread execution times** and the **system tick time**, which is the underlying number used by APIs such as `GetTickCount` that developers use to time operations in their applications. This part of the work is performed by `KeUpdateRunTime`. Before doing any of that work, however, `KeUpdateRunTime` checks whether any timers have expired.
- Because the clock fires at known interval multiples, the **bottom bits** of the current system time will be at one of 64 known positions (on an APIC HAL). Windows uses that fact to organize all driver and application timers into **linked lists** based on an **array** where each entry corresponds to a **possible multiple of the system time**. This table, called the *timer table*, is located in the `PRCB`, which enables each processor to perform its own independent timer expiration without needing to acquire a **global lock**.
- Each multiple of the system time that a timer can be associated with is called the *hand*, and it’s stored in the timer object’s **dispatcher header**.
- Therefore, to determine if a clock has expired, it is only necessary to check if there are any timers on the linked list associated with the current hand.
<p align="center"><img src="./assets/per-processor-timer-lists.png" width="400px" height="auto"></p>

- Similarly to how a driver ISR queues a DPC to defer work, the clock ISR requests a DPC software interrupt, setting a flag in the `PRCB` so that the DPC draining mechanism knows timers need expiration.
- Likewise, when updating process/thread runtime, if the clock ISR determines that a thread has expired its quantum, it also queues a DPC software interrupt and sets a different `PRCB` flag.
- Once the IRQL eventually drops down back to `DISPATCH_LEVEL`, as part of DPC processing, these two flags will be picked up.

#### Processor Selection

- A critical determination that must be made when a timer is inserted is to pick the appropriate table to use—in other words, the most optimal processor choice.
- If the timer has **no DPC associated with it**, the kernel **scans all processors** in the current processor’s group that have not been parked. If the current processor is parked, it picks the next processor in the group; otherwise, the current processor is used.
- If the timer does **have an associated DPC**, the insertion code simply looks at the target processor associated with the DPC and selects that processor’s timer table.

<details><summary>Listing System Timers:</summary>

- You can use the kernel debugger to dump all the current registered timers on the system, as well as information on the DPC associated with each timer (if any) . See the following output for a sample:
```c
   lkd> !timer
   Dump system timers
   Interrupt time: 61876995 000003df [ 4/ 5/2010 18:58:09.189]
   List Timer    Interrupt Low/High     Fire Time              DPC/thread
   PROCESSOR 0 (nt!_KTIMER_TABLE fffff80001bfd080)
     5 fffffa8003099810   627684ac 000003df [ 4/ 5/2010 18:58:10.756]
   NDIS!ndisMTimerObjectDpc (DPC @ fffffa8003099850)
   13 fffffa8003027278   272dde78 000004cf [ 4/ 6/2010 23:34:30.510]  NDIS!ndisMWakeUpDpcX
   (DPC @ fffffa80030272b8)
       fffffa8003029278   272e0588 000004cf [ 4/ 6/2010 23:34:30.511]  NDIS!ndisMWakeUpDpcX
   (DPC @ fffffa80030292b8)
       fffffa8003025278   272e0588 000004cf [ 4/ 6/2010 23:34:30.511]  NDIS!ndisMWakeUpDpcX
   (DPC @ fffffa80030252b8)
       fffffa8003023278   272e2c99 000004cf [ 4/ 6/2010 23:34:30.512]  NDIS!ndisMWakeUpDpcX
   (DPC @ fffffa80030232b8)
    16 fffffa8006096c20   6c1613a6 000003df [ 4/ 5/2010 18:58:26.901]  thread
   fffffa8006096b60
   ...
```
</details>

#### Intelligent Timer Tick Distribution

- The figure below which shows processors handling the clock ISR and expiring timers, reveals that processor 1 wakes up a number of times (the solid arrows) even when there are no associated expiring timers (the dotted arrows).

<p align="center"><img src="./assets/timer-expiration.png" width="400px" height="auto"></p>

- Because the only other work required that was referenced earlier is to update the overall system time/clock ticks, it’s sufficient to designate merely one processor as the **time-keeping processor** (in this case, processor 0) and allow other processors to remain in their **sleep state**; if they wake, any time-related adjustments can be performed by **resynchronizing** with processor 0.
- Windows does, in fact, make this realization (internally called *intelligent timer tick distribution*), and the figure below shows the processor states under the scenario where processor 1 is sleeping (unlike earlier, when we assumed it was running code).
- :arrow_forward: As the processor detects that the work load is going lower and lower, it decreases its **power consumption** (P states), until it finally reaches an idle state.
<p align="center"><img src="./assets/intelligent-timer-tick-distribution.png" width="400px" height="auto"></p>

#### Timer Coalescing

- Why ? Reducing the amount of software timer-expiration work would both help to decrease latency (by requiring less work at DISPATCH_LEVEL) as well as allow other processors to stay in their sleep states even longer (because we’ve established that the processors wake up only to handle expiring timers, fewer timer expirations result in longer sleep times).
- How ? Combine separate timer hands into an individual hand with multiple expirations.
- Assuming that most drivers and user-mode applications do not particularly care about the exact firing period of their timers :)
- Not all timers are ready to be coalesced into coarser granularities, so Windows enables this mechanism only for timers that have **marked themselves as coalescable**, either through the `KeSetCoalescableTimer` kernel API or through its user-mode counterpart, `SetWaitableTimerEx`.
- Assuming all the timers specified tolerances and are thus coalescable, in the previous scenario, Windows could decide to coalesce the timers as:
<p align="center"><img src="./assets/timer-coalescing.png" width="400px" height="auto"></p>

### Exception Dispatching

- In contrast to interrupts, which can occur at any time, exceptions are conditions that **result directly from the execution of the program that is running**.
- On the x86 and x64 processors, all exceptions have **predefined interrupt numbers** that directly correspond to the entry in the `IDT` that points to the trap handler for a particular exception.
- Table below shows x86-defined exceptions and their assigned interrupt numbers. Because the first entries of the IDT are used for exceptions, hardware interrupts are assigned entries later in the table.

|Interrupt Number|Exception|
|----------------|---------|
|0 |Divide Error|
|1 |Debug (Single Step)|
|2 |Non-Maskable Interrupt (NMI)|
|3 |Breakpoint|
|4 |Overflow|
|5 |Bounds Check|
|6 |Invalid Opcode|
|7 |NPX Not Available|
|8 |Double Fault|
|9 |NPX Segment Overrun|
|10| Invalid Task State Segment (TSS)|
|11| Segment Not Present|
|12| Stack Fault|
|13| General Protection|
|14| Page Fault|
|15| Intel Reserved|
|16| Floating Point|
|17| Alignment Check|
|18| Machine Check|
|19| SIMD Floating Point|

- The kernel traps and handles some of these exceptions **transparently** to user programs (i.e a breakpoint when a debugger is attached).
- A few exceptions are allowed to filter back, untouched, to user mode (memory-access violations).
	- **32-bit apps** can establish **frame-based** exception handlers to deal with exceptions.
		- The term frame-based refers to an exception handler’s association with a particular **procedure activation**.
		- When a procedure is invoked, a **stack frame** representing that activation of the procedure is pushed onto the stack.
		- A stack frame can have one or more exception handlers associated with it, each of which protects a particular block of code in the source program.
		- When an exception occurs, the kernel searches for an exception handler associated with the current stack frame.
		- If none exists, the kernel searches for an exception handler associated with the previous stack frame, and so on, until it finds a frame-based exception handler. If no exception handler is found, the kernel calls its **own default exception handlers**.
	- For **64-bit apps**, SEH does not use frame-based handlers.
		- Instead, a table of handlers for each function is **built into the image during compilation**.
		- The kernel looks for handlers associated with each function and generally follows the same algorithm we described for 32-bit code.
- When an exception occurs, whether it is explicitly raised by software or implicitly raised by hardware, a chain of events begins in the kernel.
	1. The CPU hardware transfers control to the kernel trap handler, which creates a **trap frame** (as it does when an interrupt occurs).
	2. The trap frame allows the system to resume where it left off if the exception is resolved.
	3. The trap handler also creates an **exception record** that contains the reason for the exception and other pertinent information.
- If the exception occurred in **kernel mode**, the exception dispatcher simply calls a routine to locate a frame-based exception handler that will handle the exception.
- If the exception occurred in **user mode**, the kernel uses a **debug object/port** in its default exception handling. The Windows subsystem has a **debugger port** and an **exception port** to receive notification of user-mode exceptions in Windows processes.
<p align="center"><img src="./assets/exception-dispatching.png" width="400px" height="auto"></p>

- If the process has no debugger process attached or if the debugger doesn’t handle the exception, the exception dispatcher switches into user mode, copies the trap frame to the user stack formatted as a `CONTEXT` data structure, and calls a routine to find a SE/VE handler. If none is found or if none handles the exception, the exception dispatcher switches back into kernel mode and calls the debugger again to allow the user to do more debugging. This is called the **second-chance notification**.
- If the debugger isn’t running and no user-mode exception handlers are found, the kernel sends a message to the exception port associated with the thread’s process. The exception port gives the environment subsystem the opportunity to translate the exception into an environment-specific signal or exception.
- If the kernel progresses this far in processing the exception and the subsystem doesn’t handle the exception, the kernel sends a message to a **systemwide error port** that *Csrss* uses for *Windows Error Reporting* (WER), and executes a default exception handler that simply terminates the process whose thread caused the exception (often by launching the *WerFault.exe*).

#### Windows Error Reporting (WRE)

- WER is a sophisticated mechanism that automates the submission of both user-mode process crashes as well as kernel-mode system crashes.
- When an unhandled exception is caught by the unhandled exception filter, it builds context information (registers and stack) and opens an **ALPC port** connection to the WER service.
- WER service begins to analyze the crashed program’s state and performs the appropriate actions to notify the user, in most cases this means launching the *WerFault.exe* program.
- In default configured systems, an error report (a minidump and an XML file with various details) is sent to Microsoft’s online crash analysis server.
- WER performs this work **externally** from the **crashed thread** if the unhandled exception filter itself crashes, which allows any kind of process or thread crash to be logged and for the user to be notified.
- WER service uses an ALPC port for communicating with crashed processes. This mechanism uses a systemwide error port that the WER service registers through `NtSetInformationProcess` (which uses `DbgkRegisterErrorPort`). As a result, all Windows processes now have an error port that is actually an ALPC port object registered by the WER service.
- Foldable.: The kernel, which is first notified of an exception, uses this port to send a message to the WER service, which then analyzes the crashing process. This means that even in severe cases of thread state damage, WER will still be able to receive notifications and launch WerFault.exe to display a user interface instead of having to do this work within the crashing thread itself . Additionally, WER will be able to generate a crash dump for the process, and a message will be written to the Event Log. This solves all the problems of silent process death: users are notified, debugging can occur, and service administrators can see the crash event.

### System Service Dispatching

#### System Service Dispatching

- On x86 processors prior to the *Pentium II*, Windows uses the `int 0x2e` instruction, which results in a trap. Windows fills in entry 46 in the IDT to point to the system service dispatcher.
- The trap causes the executing thread to transition into kernel mode and enter the system service dispatcher.
- `EAX` register indicates the **system service number** being requested.
- `EDX` register points to the **list of parameters** the caller passes to the system service.
- To return to user mode, the system service dispatcher uses the `iret` (interrupt return instruction).
- On x86 *Pentium II*+, Windows uses the `sysenter` instruction, which Intel defined specifically for **fast system service dispatches**. 	- To support the instruction, Windows stores at boot time the **address** of the kernel’s **system service dispatcher** routine in a **machine-specific register** (MSR) associated with the instruction.
	- EAX and EDX plays the same role as for the INT instruction.
	- To return to user mode, the system service dispatcher usually executes the `sysexit` instruction.
	- (In some cases, like when the **single-step flag** is enabled on the processor, the system service dispatcher uses the `iret` instead because `sysexit` does not allow returning to user-mode with a different `EFLAGS` register, which is needed if `sysenter` was executed while the trap flag was set as a result of a **user-mode debugger** tracing or stepping over a system call.)
- On the x64 architecture, Windows uses the `syscall` instruction, passing the **system call number** in the `EAX` register, the **first four parameters** in **registers**, and any parameters beyond those four on the **stack**.

<details><summary>Locating the System Service Dispatcher:</summary>

- To see the handler on 32-bit systems for the `interrupt 2E` version of the system call dispatcher:
```c
   lkd> !idt 2e
   Dumping IDT:
   2e:    8208c8ee nt!KiSystemService
 ```
2. To see the handler for the `sysenter` version, use the `rdmsr` debugger command to read from the MSR register 0x176, which stores the handler:
```c
   lkd> rdmsr 176
   msr[176] = 00000000'8208c9c0

    lkd> ln 00000000'8208c9c0
           (8208c9c0)   nt!KiFastCallEntry
```
- If you have a 64-bit machine, you can look at the service call dispatcher by using the `0xC0000082` MSR instead. You will see it corresponds to `nt!KiSystemCall64`:
```c
   lkd> rdmsr c0000082
   msr[c0000082] = fffff800'01a71ec0
   lkd> ln fffff800'01a71ec0
   (fffff800'01a71ec0)   nt!KiSystemCall64
 ```
- You can disassemble the `KiSystemService` or `KiSystemCall64` routine. On a 32-bit system, you’ll eventually notice the following instructions:
```c
   nt!KiSystemService+0x7b:
   8208c969 897d04          mov     dword ptr [ebp+4],edi
   8208c96c fb              sti
   8208c96d e9dd000000      jmp     nt!KiFastCallEntry+0x8f (8208ca4f)
```
- Because the actual system call dispatching operations are common regardless of the mechanism used to reach the handler, the older interrupt-based handler simply calls into the middle of the newer sysenter-based handler to perform the same generic tasks.
- In 32-bit Windows:
```c
0:000> u ntdll!NtReadFile
ntdll!ZwReadFile:
77020074 b802010000		mov     eax,102h
77020079 ba0003fe7f		mov     edx,offset SharedUserData!SystemCallStub (7ffe0300) // system service dispatch code SystemCallStub member 											// of the KUSER_SHARED_DATA
7702007e ff12			call    dword ptr [edx]
77020080 c22400			ret 	24h
77020083 90				nop
```
- Because 64-bit systems have only one mechanism for performing system calls, the system service entry points in *Ntdll.dll* use the syscall instruction directly:
```c
ntdll!NtReadFile:
00000000'77f9fc60 4c8bd1
00000000'77f9fc63 b810200000
00000000'77f9fc68 0f05             syscall
00000000'77f9fc6a c3               ret
```
</details>

#### Kernel-Mode System Service Dispatching

- The kernel uses the system call number to locate the system service information in the **system service dispatch table (SSDT)**.
- On 32-bit systems, this table is similar to the interrupt dispatch table described earlier in the chapter except that each entry contains a **pointer to a system service** rather than to an interrupt-handling routine.
- On 64-bit systems, the table is implemented slightly differently—instead of **containing pointers to the system service**, it contains **offsets relative to the table itself**.
<p align="center"><img src="./assets/system-service-exceptions.png" width="400px" height="auto"></p>

- `KiSystemService`, copies the caller’s arguments from the thread’s user-mode stack to its kernel-mode stack and then executes the system service.
- The kernel knows how many stack bytes require copying by using a second table, called the **argument table**, which is a byte array (instead of a pointer array like the dispatch table), each entry describing the number of bytes to copy.
- On 64-bit systems, Windows actually encodes this information within the service table itself through a process called **system call table compaction**.
- The **previous mode** is a value (kernel or user) that the kernel saves in the thread whenever it executes a trap handler and identifies the privilege level of the incoming exception, trap, or system call.
	- If a system call comes from a driver or the kernel itself, the probing and capturing of parameters is skipped, and all parameters are assumed to be pointing to valid kernel-mode buffers.
	- If it is coming from user-mode, the buffers **must be probed**.
- Drivers should call the Zw* version (which is a trampoline to the Nt* system call) of the system calls, because they set the previous mode to kernel. Instead of generating an interrupt or a `sysenter`, which would be slow and/or unsupported, they build a fake interrupt stack (the stack that the CPU would generate after an interrupt) and call the `KiSystemService` routine directly, essentially emulating the CPU interrupt.
- Windows has two system service tables, and third-party drivers cannot extend the tables or insert new ones to add their own service calls. On 32-bit and IA64 versions of Windows, the system service dispatcher locates the tables via a pointer in the **thread kernel structure**, and on x64 versions it finds them via their **global addresses**.
	- The system service dispatcher determines which table contains the requested service by interpreting a **2-bit** field in the 32-bit system service number as a table index.
	- The low **12 bits** of the system service number serve as the index into the table specified by the table index.

<p align="center"><img src="./assets/system-service-tables.png" width="400px" height="auto"></p>

#### Service Descriptor Tables

- A primary default array table, `KeServiceDescriptorTable`, defines the core executive system services implemented in *Ntosrknl.exe*.
- The other table array, `KeServiceDescriptorTableShadow`, includes the Windows USER and GDI services implemented in *Win32k.sys*.
- The difference between System service dispatching for Windows kernel API and a USER/GDI API call.
<p align="center"><img src="./assets/system-service-dispatching.png" width="400px" height="auto"></p>

<details><summary>Mapping System Call Numbers to Functions and Arguments:</summary>

- The `KeServiceDescriptorTable` and `KeServiceDescriptorTableShadow` tables both point to the same array of pointers (or offsets, on 64-bit) for kernel system calls, called `KiServiceTable`.
- On a 32-bit system, you can use the kernel debugger command `dds` to dump the data along with symbolic information.
```c
lkd> dds KiServiceTable
     820807d0  821be2e5 nt!NtAcceptConnectPort
     820807d4  820659a6 nt!NtAccessCheck
     820807d8  8224a953 nt!NtAccessCheckAndAuditAlarm
     820807dc  820659dd nt!NtAccessCheckByType
     820807e0  8224a992 nt!NtAccessCheckByTypeAndAuditAlarm
     820807e4  82065a18 nt!NtAccessCheckByTypeResultList
     820807e8  8224a9db nt!NtAccessCheckByTypeResultListAndAuditAlarm
     820807ec  8224aa24 nt!NtAccessCheckByTypeResultListAndAuditAlarmByHandle
     820807f0  822892af nt!NtAddAtom
```
- On a 64-bit system, the base of the pointer is the `KiServiceTable` itself, so you’ll have to dump the data in its raw format with the `dq` command:
```c
lkd> dq nt!KiServiceTable
	fffff800'01a73b00  02f6f000'04106900 031a0105'fff72d00
```
</details>