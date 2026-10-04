# Chapter 8: Scheduling

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

```rust
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
        println!("[TIMER] The timer just went off!!!!")
        irq_sources &= !(InterruptSource::PhysicalNonSecureTimer as u32);
    }

    if irq_sources > 0 {
        println!("Other Unhandled IRQ sources pending: {:#x}", irq_sources).unwrap();
        unhandled_exception!(ctx);
    }
}
```

And this completes our exception handling pipeline. An IRQ exception occurs, our rust exception handler catches it, identifies it as IRQ, and if it came from the Non Secure Physical Timer going off, it is going to print the following message:

```
[TIMER] The timer just went off!!!!
[TIMER] The timer just went off!!!!
...
```

Note that you will not get just a single print. Like we discussed in the last chapter, the exception will keep being triggered every time the timer's counter updates and notices the trigger condition is true. So we need to also disable the timer inside the handler.

Let's add this to the top of `exceptions.rs`.
```rust
use crate::kernel::timer::PhysicalTimer;
```

Then add the following to our code, you can either mask the timer interrupt, or disable it, or you can just move the timer deadline forward into the future.

```rust
    if irq_sources & (InterruptSource::PhysicalNonSecureTimer as u32) != 0 {
        PhysicalTimer::set_seconds(1); // disable irq by immedaitely scheduling it into the future
        PhysicalTimer::disable();
        println!("[TIMER] The timer just went off!!!!")
        irq_sources &= !(InterruptSource::PhysicalNonSecureTimer as u32);
    }
```

And now finally now when the timer goes off, you will see a single output message and the timer will be safely disabled. No more unnecessary timer exceptions!

```
[TIMER] The timer just went off!!!!
```

You may try this out by setting the timer in the kernel rust main. And seeing the output go off. Similar to how it was instructed in the last chapter.

## Starting the pulse

Now, finally our timer pipeline is complete. We can see a timer go off, and get turned off appropriately. Now, during scheduling what we're going to need is to make the timer go off periodically at a fixed time period. This will end up becoming the heartbeat of our scheduler. Let's say we choose the time period to be 10ms. Then after every 10ms our scheduler will be triggered, whereby it will update saved contexts and then schedule the next appropriate process according to some _scheduling algorithm_.

To make the timer go off periodically it is very simple. You just need to make it so that when the timer goess off, then while handling the IRQ, we also set the timer again to go off in the exact same amount of time as before. 

```rs
    if irq_sources & (InterruptSource::PhysicalNonSecureTimer as u32) != 0 {
        PhysicalTimer::set_seconds(1); // disable irq by immedaitely scheduling it into the future
        PhysicalTimer::disable();
        println!("[TIMER] The timer just went off!!!!")
        PhysicalTimer::set_milliseconds(10);
        PhysicalTimer::enable();
        irq_sources &= !(InterruptSource::PhysicalNonSecureTimer as u32);
    }
```

And now, this timer is going to keep going off every 10ms.

## Process B

It's pretty annoying to not just get into it. But there's still one more thing we need to setup before we actually implement a scheduler. Can you guess it? 

We need another process of course! How will we see our scheduler work if there is no other processes to schedule? If there was only a single process running, we'd just be seeing it get interrupted by the timer and then going back to running again like nothing changed.

For this, we will follow the exact same procedure we did for our first process. In your user project, go ahead and create a new Rust program file under the `bin/` directory. Same as our original user program. Let's name it something like `b.rs`.

```rust
// process B just for the sake of testing the scheduler with init and process B

#![no_std]
#![no_main]

use user::{entry, println};

fn main() {
    println!("hello this code is running in process B!").unwrap();
    
    let mut x = 1;
    println!("x = {}", x).unwrap();
    x += 1;
    println!("x = {}", x).unwrap();
    
    println!("process B is done working, it will now loop forever.").unwrap();
    loop {println!("This is b looping forever!").unwrap(); core::hint::spin_loop();}
}

entry!(main);
```

The code could look like something you can see above. It's just a simple program that alos prints something to the output. The only difference is that it should output something which is different from the original `init` progran we previously wrote. If they both output the same thing it will be difficult to tell their output apart during scheduling tests.

For the init user progran, we're gonna use something like the following:

```rust
#![no_std]
#![no_main]

use user::{entry, println};

fn main() {
    println!("hello this code is running in the init program!").unwrap();

    let mut x = 1;
    println!("x = {}", x).unwrap();
    x += 1;
    println!("x = {}", x).unwrap();
    
    println!("init program is done working, it will now loop forever.").unwrap();
    loop {println!("This is init looping forever!").unwrap(); core::hint::spin_loop();}
}

entry!(main);
```

Both are essentially the same, only the output identifies which program is doing the printing. The point is when we test out scheduling, the running program's output will tell us which program is running.

Once both programs are written, you have to compile them to a `.bin` file according to the previous chapters. You may remember that the linker script we used was:

```ld
ENTRY(_start)

SECTIONS
{
    /* once memory virtualization works, i will change this to start at 0x00000000 */
    . = 0x200000;
(...)
```

However, since both programs will need to be loaded into memory at the same time, you cannot load both the programs at the same memory address that literally does not make any physical sense. We will need to load `b` program at some other memory address. Let's say `0x500000`. Therefore, first compile with the original linker script. Then use the `init` ELF produced here to get `init.bin`. But then for `b.bin` image, change linker script to start address layout at `0x500000`. Now compile again, and then use this newly produced `b` ELF for producing the `b.bin` image. This way you have both images which are meant to be placed at different memory addresses. 

You can get the entry point for `b.bin` the same way you got for `init.bin` back in chapter 4. And then in rust main of the kernel, you can use the load_process function we implemented back in chapter 6.

```rust
    let process_a_image: &'static [u8] = include_bytes!("user/init.bin");
    let process_b_image: &'static [u8] = include_bytes!("user/b.bin");

    // note from ch6 implementation: load process does not include enter_user() execution
    load_process("init", 0, process_a_image, 0x200000, 0x200274);
    load_process("process b", 0, process_b_image, 0x500000, 0x500334); 
                                                         // 0x500334 is the entry point
```

Now with that, we have two separate processes loaded into our memory at the same time. Ready to execute. Both are saved in the process table thanks to our `load_process` function. Now all we need to do is jump to one of their entry point addresses with level `EL0`.

That is something our scheduler will do. 


## Scheduler

Finally, we can use the pipeline we have setup and use it to create a working scheduler. 

### Architecture

Now, you probably already have a decent idea  of what we're going to do. But let's go over it before we jump into implementation.

What we have to do is set a timer before running a process. Then when the timer goes off then:

- Save exception context of the process as a process context to the process table.
- Go through all the processes in the process table which are waiting to be run.
- Choose the next process whose turn it is to run next.
- Overwrite the exception context with the process context of the chosen process.
  - In essence we are loading the context of this process
- Then return from exception will cause the chosen process to run from where its context dictates.

In essence, our scheduler is going to hijack the exception handling pipeline to perform context switching. The exception context is what dictates which instruction to return to, what exception level to return to, the process state, etc. So by simply switching the exception context we can switch the process which is going to run after exception return. 

Functionality wise this will be achieved by writing a Scheduler class. it will have some method like `Scheduler::schedule_next(&mut ExceptionContext)` which will do all the procedure of choosing next process and overwriting the exception context with said process's context. It should also havev a method for saving last running process's context to corresponding process table entry before the exception context is overwritten.

So something like:

```
(PSEUDOCODE)

(in exception handler)

if timer went off in EL0: (i.e. timer interrupted user program running)
    -> the exception context holds the context of the running 
       process just before it was interrupted.
    -> so save that as process context in said process's 
       process table entry.
    -> change said process's stage from RUNNING to READY.
    -> now choose which READY process to run next from the process table.
    -> overwrite exception context with said process's process context.
    -> set that new process's state to RUNNING
    -> reset the timer (so pulse continues)
    -> return from exception.   
```

We need to create methods in a new Scheduler class that can be called to achieve all these procedures. 

Let's start off by creating a new module for the scheduler. `srx/kernel/scheduler.rs`. 

```rs
pub static mut CURRENT_PROCESS: usize = 0; // last scheduled process index in process table
pub const TIMESLICE_MILISECONDS: u64 = 9; 

// we implement xv6 similar round robin

pub struct Scheduler;
```

`CURRENT_PROCESS` is just a variable which will keep track of which process the scheduler scheduled last. We will update this appropriately whenever schedule a new process. Initial value is not important as the scheduler will set it appropriately the moment it is called to schedule the first process.

And now, let's first start off by moving the timer going off handling part into the scheduler.

```rs
impl Scheduler {

    fn reset_timer() {
        PhysicalTimer::set_milliseconds(TIMESLICE_MILISECONDS);
        PhysicalTimer::enable();
    }

}
```

Next up, let's write the function which will be used to mark last running process from `RUNNING` to `READY`. 

```rs
    pub fn timeslice_up() {
        // updating current processs from running to ready.
        unsafe {
            if let Some(current_process) = &mut PROCESS_TABLE[CURRENT_PROCESS] {
                current_process.set_state(ProcessState::Ready);
            } else {
                panic!("Current process disappeared for unaccounted reason!");
            }
        }

        Self::reset_timer();
    }
```

Therefore when the timer goes off, we will call `Scheduler::timeslice_up()`. Then we will do the rest of the procedure of saving context and scheduling next process.

Next up, the method for saving the context of the last running process to the process table: 

```rs
    pub fn update_last_running_pctx(new_pctx: &ProcessContext) {
        unsafe {
            if let Some(current_process) = &mut PROCESS_TABLE[CURRENT_PROCESS] {
                current_process.set_pctx(*new_pctx);
            } else {
                panic!("Last running process disappeared for unaccounted reason!");
                // panic, because this function is going to only exclusively called
                // in handle_exception_el1. and ONLY in the case when the exception
                // came from EL0. which would be our last scheduled user process.
                // so if for some mysterious reason the process just disappeared
                // after an exception came from it, we might want kernel to scream.
            }
        }
    }
```

Note that it accepts `ProcessContext` rather than `ExceptionContext`. You could also make it directly accept `ectx` instead of `pctx`, but I choose to do it this way to make it more versatile outside of exception handling pipelines where ectx is not available.

To catchup, so far our pipeline is looking like:

```
PSEUDOCODE

exception_handler:
    
    if exception occured from EL0: (i.e. user process interrupted) {
        
        read value of sp_el0 register (because it is not saved in exception_ctx) 
        let new_pctx = ProcessContext::from_ectx(exception_ctx, sp_el0);

        Scheduler::update_last_running_pctx(&new_pctx);        

    }

    if exception was timer irq {
        // turn off timer (so it doesn't keep going off)

        Scheduler::timeslice_up();
    
        // choose next process to schedule
        // overwrite exception.ctx to said process.ctx
    }
```

Next up, we need to write the code which will do the last two steps. That is, scheduling the next process.

Let's write a program which will simply choose the next process. It will merely return the index of the chosen process in the process table.

```rust
    fn choose_next_process() -> Option<usize> {
        unsafe {
            // if current progress has not finished its time slice, continue it
            if let Some(current_process) = PROCESS_TABLE[CURRENT_PROCESS] {
                if current_process.state == ProcessState::Running {
                    return Some(CURRENT_PROCESS as usize);
                }
            }
            // otherwise scan forward circularly for next process
            for i in 1..(MAX_PROCESSES+1) { // +1, so if no other processes are found, it will circle back to current process
                let idx = (CURRENT_PROCESS + i) % MAX_PROCESSES;
                if let Some(process) = PROCESS_TABLE[idx] {
                    if process.state == ProcessState::Ready {
                        CURRENT_PROCESS = idx;
                        return Some(idx);
                    }
                }
            }
        }
        None
    }
```

Firstly let's understand the first if-statement. It merely states that if the last scheduled process is still in RUNNING state, then just choose it again. This is to ensure we don't choose a different process to run while there's already one supposed to be running. You might think this is redundant since we make sure to set the last running process as READY in the `timeslice_up` method. However, adding this guard helps make the code more safe, and more flexible if this function needs to be called in some different situation in the future. 

Next up, the first for loop merely loops through all the process table entries. If there is an entry which is of "READY" state, it is chosen. It also makes sure to start scan from `CURRENT_PROCESS_INDEX + 1`. So the same process is not scheduled again. It scans circularly, up till `CURRENT_PROCESS_INDEX`. So if no process is found, it will ultimately circle back and choose the same process again.

Lastly if there's no processes ready to be run, we return `None`. 

Now, let's write a function to update the exception context according to process choice.

```rust
    pub fn schedule_next(ectx: &mut ExceptionContext) {
        if let Some(next_process) = Self::choose_next_process() {
            Self::load_pctx(next_process, ectx);
        } else {
            println!("[SCHEDULER] No process to schedule!").unwrap();
            the_end();
        }
    }

    fn load_pctx(pidx: usize, ectx: &mut ExceptionContext) {
        unsafe {
            if let Some(process) = &mut PROCESS_TABLE[pidx] {
                CURRENT_PROCESS = pidx;
                process.set_state(ProcessState::Running);
                ectx.update_from_pctx(&process.pctx);
                core::arch::asm!("msr SP_EL0, {sp}", sp = in(reg) process.pctx.sp);
            } else {
                panic!("in Scheduler::load_pctx(), process not found!");
                // panic, because this function is only called in schedule() and ONLY after choose_next_process() returns Some(pidx). so if for some mysterious reason the process just disappeared after being chosen, we might want kernel to scream.
            }
        }
    }
```

This is the function which will be called in the exception handler. The `ExceptionContext` will be passed to it. It uses previous written function to choose the next process, and then loads appropriate process context into the exception context. 

The `SP_EL0` register is not included in the exception context so at every step we're having to set it directly using `asm!` macro. Since the exception handler does not update it. You could however incorporate it into the exception handling pipeline as an exercise. We will do it in this book in a future chapter. 

Also note the function `the_end()`. It stands to reason that if theres no more processes left to schedule, It must mean all processes have terminated and finished working. Therefore if we sense that no process is available to schedule, we do the following:

```rust
// function is defined in main.rs
pub fn the_end() -> ! {
    println!("All processes have completed/terminated.").unwrap();
    println!("There is nothing left to do. You may power off your device now.").unwrap();
    loop { core::hint::spin_loop(); }
}
```

And with that we have all the methods and functions we really need!

We can finally put them in the places they belong.

Firstly, let's clean up the exception handler for the timer IRQ.

Let's create a new function in the `PhysicalTimer` struct itself to handle IRQs.

```rs
impl PhysicalTimer {
    pub fn handle_irq(ctx: &mut ExceptionContext) { 
        PhysicalTimer::set_seconds(1); // disable irq by immedaitely scheduling it into the future
        PhysicalTimer::disable();

        Scheduler::timeslice_up();
        Scheduler::schedule_next(ctx);
    }
}
```

And now, the original IRQ handler can be written as:

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

Look closely, this is almost our entire pipeline! if a timer goes off, the `PhysicalTimer::handle_irq(ctx)` handles the following:
- disable timer 
- `Scheduler::timeslice_up()`: 
    - mark `RUNNING` process as `READY`
    - reset timer
- `Scheduler::schedule_next(ctx)`:
    - choose next process to schedule
    - overwrite exception `ctx` with chosen process's context

There is still one thing left however, can you guess what it is? 

Yes, it is the saving of exception context of last running process before we overwrite it.

We will do that part as the first thing in the exception handler itself. The idea is no matter what the cause was, if a user program was interrupted, update its process context. This is so the process context is as up-to-date as possible for any kernel procedure referring to it. 

Firstly a function to check if exception occured in EL0:

```rust
fn was_from_user_el0(ctx: &ExceptionContext) -> bool {
    ctx.esource == ExceptionSource::_EL064 
          || ctx.esource == ExceptionSource::_EL032
}
```

And now, we add the following to our exception handler: 

```rust

// called by `exceptions.s`
#[unsafe(no_mangle)]
pub extern "C" fn handle_exception_el1(ctx: &mut ExceptionContext) {
    
    // if it came from EL0, then we need to update PCB of the process interrupted.
    if was_from_user_el0(ctx) {
        let sp_el0: u64;
        unsafe {
            // reading sp_el0
            core::arch::asm!(
                "mrs {val}, sp_el0",
                val = out(reg) sp_el0,
                options(nostack, preserves_flags)
            )
        }
        let new_pctx = ProcessContext::from_ectx(ctx, sp_el0);

        Scheduler::update_last_running_pctx(&new_pctx);
    }

    // println!("An exception has been detected :D").unwrap();
    
    // handling the exception based on the type and source.
    match ctx.etype {
        // Rest is same as before...
```

It's very self explanatory. If exception occured in EL0, then create a new `ProcessContext` object from exception context + SP_EL0 register value. And call `Scheduler::update_last_running_pctx` to write said process context to the appropriate index in the process table. 

Congratulations! Just like that we have officially finished implementing the entire scheduler pipeline! Every time the timer goes off, the scheduler will kick in and switch the context to the next process to run. 

## Starting the Scheduler Loop

Now, to trigger the pipeline to start running, all we have to do is load our processes in the process table using the `load_process` function. And then set the physical timer for the first time. Wait for it to go off and watch the scheduler go!

```rust

// in kernel rust main

    PhysicalTimer::init_irq();
    Interrupts::daif_unmask_all();

    let process_a_image: &'static [u8] = include_bytes!("user/init.bin");
    let process_b_image: &'static [u8] = include_bytes!("user/b.bin");

    load_process("init", 0, process_a_image, 0x200000, 0x200274);
    load_process("process b", 0, process_b_image, 0x500000, 0x500334);

    println!("Starting the scheduler!").unwrap();
    PhysicalTimer::set_seconds(1);
    PhysicalTimer::enable();

```

Build, and try running it on QEMU/RPi. You'll see an output such as follows:

```bash
$ qemu-system-aarch64 -M raspi3b -kernel kernel8.img -serial null -serial stdio

Starting the scheduler!
hello this code is running in process B!
hello this code is running in the init program!
x = x = 1
x = 2
process B is done working, it will now loop forever.
This is b looping forever!
This is b looping forever!
This is b looping forever!
This is b looping forever!
1
x = 2
init program is done working, it will now loop forever.
This is init looping forever!
This is init looping forever!
This is init looping forever!
This is init looping forever!
This is b looping forever!
This is b looping forever!
This is b looping forever!
This is b looping forever!
This is b looping forever!
This is init looping forever!
This is init looping forever!
This is init looping forever!
This is init looping forever!
This is init looping forever!
This is b looping forever!
This is b looping forever!
(...truncated)
```

## Conclusion

Therefore, the output we see shows programs `init` and `b` running alternatively as separate processes. We see a few lines of output from process `init` and a few lines from `b` then again from `init` and so on. Therefore the two process are being switched between repeatedly. Therefore the scheduling procedure has been implemented successfully!

If you create more programs, you can include their binary image the same way as `init.bin` or `b.bin` and load them using `load_process`. Then that new process will also be scheduled according to the *Round Robin* algorithm that we have implemented. Trying this out is left as an exercise for the reader.

## Final codes

Snapshot of the state of the project so far can be found at: 

[github.com/ZackyGameDev/AtOS/tree/aef3fa0c404b7423a9ef92770c3f12e91604708e](https://github.com/ZackyGameDev/AtOS/tree/aef3fa0c404b7423a9ef92770c3f12e91604708e)