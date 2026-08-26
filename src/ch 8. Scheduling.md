## (Below chapter is currently under work)

# Scheduling

We have already gone over how we can get an interrupt to be triggered in our kernel at a scheduled time duration. We are now going to utilize this new feature to implement a process scheduling system. As we know, your computer CPU can usually only run a single process at a time for *each CPU core*. It only has one program counter register and is able to track only a single "thread" of instructions in memory at once. Then how is it possible that processes in memory seem to be able to run over a hundred processes simultaneously? 

The answer; like we briefly discussed during chapter 6, is context switching. Since the introduciton in chatper 6, you already know very briefly what context switching and scheduling actually is. In this chapter we are now going to elaborate on it and go deeper on how these concepts will be implemented architecturally.

Firslty, let's aim for learning and implementing a context switching system for only two processes. Then once we have two processes working *concurrently* because of the scheduling, then we are going to talk about scheduling unknown number of processes.  

## The Timer Pulse

Now, we already know that the way our OS runs a user process, is by doing `ERET` over to the user program instructions with `EL0` level. After that, the CPU is stuck running instructions from the user program's instructions data. But now that the user program is running on the CPU, how could the kernel run the instructions it needs to run for the context switching procedure? How can the kernel run on the CPU while some user program is already running, that too in a different exception level? We need a way to force the CPU to leave user program execution right where it is, and come back to running kernel instructions in `EL1`. 

The standard way to achieve this is by exploiting the *Exception Handler*. According to the offical ARM manual [here](https://support.arm.com/documentation/102670/0301/Programmers--model/Armv8-R-AArch64-architecture-concepts/Exception-levels#:~:text=There%20is%20no%20exception%20handling%20at%20level%20EL0.%20Exceptions%20must%20be%20handled%20at%20a%20higher%20Exception%20level.), it is written:

> There is no exception handling at level EL0. Exceptions must be handled at a higher Exception level.

This means that whenever an exception occurs and the CPU is in EL0, then without fail the exception level is always raised. By default, it is raised to EL1. It may also raise to EL2 if you have configured it that way. But in our design the kernel is intended to run at EL1. Thus, an easy way to gain control back from the user program, back to the kernel would be if an exception were to occur in the middle of the user program's execution. It would trigger the exception handler. Not only does it give control back to the kernel in EL1, all CPU context is conveniently saved in the Exception Context. Which we can save as the process's context while context switching.

We have already seen this kind of procedure where the control goes from a running EL0 process to the kernel by an exception. it was during chapter 5 on syscalls! Syscalls also work by this same logic. The user program causes an exception, which causes exception handler to fire in EL1. However, the flaw with this is that in this case the exception has to be manually caused by the user process using the `svc` instruction. This is not optimal as for scheduling, we need a way for an exception to occur without the contribution from the user program.

This is where the timer we implemented last chapter comes in! It is a very simple idea. Before `eret` to the user program, we are simply going to set the timer to go off after a fixed amount of time. This way, even after the program starts running in EL0, an exception will be caused automatically by the timer component going off. Then, in the kernel's exception handler we can simply monitor if the exception came from the timer or not. And handle it accordingly from there. 

Once the timer goes off in EL0, and our exception handler identifies it, we can simply set the timer again before returning from exception. This way, we can have a scheduler pulse going on. 

Now, before we dive deeper into the scheduler architecture, let us first make the exception handler be able to catch and identify the timer exception. Since right now, the timer exception is going to appear to the handler as an unhandled IRQ exception. 

## Identifying IRQ Exceptions

In this section we're simply going to introduce a branch for the exception handling. In this new branch the exception handler will identify that the current exception came from an IRQ interrupt. It is also going to identify that among all the hardware events, it came from the timer event. 

This process is similar to how we identified `svc` caused exception back in chapter 5 on syscalls. In it, firstly we narrowed the exception down to SYNC type from the exception context. And then we narrowed it down to an `svc` based exception through the information encoded in the exception syndrome register `ESR_EL1`.

The process for a timer exception is the same. You know it is an IRQ from the exception context. However for narrowing it down further, instead of reading from the exception syndrome register, you need to read from a different register. This register is in fact located in the interrupt handler QA7. We can find it in [the documentation](https://github.com/Tekki/raspberrypi-documentation/blob/master/hardware/raspberrypi/bcm2836/QA7_rev3.4.pdf) we refered to in the last chapter. In it open topic **4.10 Core interrupt sources**. In it, it depicts four registers for each of the four cores in the CPU. It has 18 bits of data, where each field is associated with one possible IRQ source. When an IRQ occurs, associated field in this register is set to 1. The CPU can then read this register and know which source an IRQ may have come from. 

<img width="642" height="450" alt="image" src="https://github.com/user-attachments/assets/2fde94ff-fa81-48c6-8737-a46cb62eaec7" />

There's also four congruent registers which serve this exact same purpose but for FIQs. 

### Abstraction 

Let's quickly introduce a basic abstraction for this in our `Interrupts` struct.

```rust
// Core interrupt sources
pub const CORE_0_IRQ_SRC:       *const u32 = (QA7_BASE + 0x60) as *const u32;
pub const CORE_0_FIQ_SRC:       *const u32 = (QA7_BASE + 0x70) as *const u32;

#[repr(u32)]
#[derive(Copy, Clone, Debug)]
pub enum InterruptSource {
    PhysicalSecureTimer = 1 << 0,
    PhysicalNonSecureTimer = 1 << 1,
    HypervisorTimer = 1 << 2,
    VirtualTimer = 1 << 3,
    Mailbox0 = 1 << 4,
    Mailbox1 = 1 << 5,
    Mailbox2 = 1 << 6,
    Mailbox3 = 1 << 7,
    GPU = 1 << 8,
    PMU = 1 << 9,
    AXI = 1 << 10,
    LocalTimer = 1 << 11,
    PeripheralInterrupt = 0x3f << 12, // last 6 bits are for peripheral interrupts
    // we don't know yet how they are implemented and used so for now i just wrote it this way
    // so (irq_sources_register_value | PeripheralInterrupt) would give you the entire peripheral
    // interrupts field.
}
```

Then, inside the struct implementation:

```rust
    pub fn pending_irq() -> u32 {
        unsafe { read_volatile(CORE_0_IRQ_SRC) }
    }

    pub fn pending_fiq() -> u32 {
        unsafe { read_volatile(CORE_0_FIQ_SRC) }
    }

    pub fn is_irq_pending(source: InterruptSource) -> bool {
        (Self::pending_irq() & (source as u32)) != 0
    }

    pub fn is_fiq_pending(source: InterruptSource) -> bool {
        (Self::pending_fiq() & (source as u32)) != 0
    }
```

Now, our interrupts abstraction is ready to handle queries related to IRQ and FIQ identification. We can now go on and create a new branch in the exception handler for an IRQ, and inside it for an IRQ which came from the NSP timer.

### Implementation

Inside `exceptions.rs`
```rust
use crate::kernel::interrupts::{Interrupts, InterruptSource};
```
And then afterwards, inside the main exception handler function, add a new `match` branch for IRQ type exceptions.

```
// called by `exceptions.s`
#[unsafe(no_mangle)]
pub extern "C" fn handle_exception_el1(ctx: &mut ExceptionContext) {
    
    // handling the exception based on the type.
    match ctx.etype {
        ExceptionType::_SYNC => handle_sync_exception(ctx),
        ExceptionType::_IRQ  => handle_irq_exception(ctx),
        _ => unhandled_exception!(ctx),
    }

}
```

Then we will create the needed `handle_irq_exception(ctx)` function as follows:

```rust
fn handle_irq_exception(ctx: &mut ExceptionContext) -> () {
    let mut irq_sources: u32 = Interrupts::pending_irq();

    if irq_sources & (InterruptSource::PhysicalNonSecureTimer as u32) != 0 {
        PhysicalTimer::handle_irq(ctx);
        irq_sources &= !(InterruptSource::PhysicalNonSecureTimer as u32);
    }

    if irq_sources > 0 {
        println!("Other Unhandled IRQ sources pending: {:#x}", irq_sources).unwrap();
        unhandled_exception!(ctx);
    }
}
```
