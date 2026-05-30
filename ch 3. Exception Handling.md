# below document is currently under progress

# Hardware Exceptions

You may already be familiar with what an exception is in programming. It is when you write some code which does something it is not supposed to do and your computer screams at you. Stuff like dividing by zero, using a variable that was never defined, etc.

However, even on hardware level there are some actions which inherently you are not supposed to do. For example if you tell the CPU to read from some memory address which doesnt even exist. Such hardware exceptions are dealt with differently than the exceptions in programming languages.

In programming languages, when a exception occurs, it is usually sent to the exception handler. The exception handler is provided information about where the exception occured and what kind, and it kindly lets you know in the console/terminal output. 

In hardware, when an exception occurs, an **interupt** occurs. Which basically means whatever PC the CPU was executing at is immediately paused. the value of the PC is stored, and then the CPU *jumps* to a different address in memory, where it expects to find instructions which-- if the CPU executes-- will handle the exception.

In simpler terms, you have to plan beforehand how you want an hardware exception to be dealt with. You have to write code for it, and compile it to machine language that can be directly executed by the CPU. And then you have to put all that code in memory, and you have to tell your CPU at which address in your memory you have placed said code. So now whenever an exception occurs, the CPU will make a backup of current PC register value, and change PC to the address where your *exception handler* code is. 

Now, naturally you might want to handle different kinds of exceptions differently. So you have to do this process of writing handler code and telling your CPU where it is, for each kind of exception. Telling your CPU where the handler is, is as simple as simply writing the handler code starting address to a certain register of the CPU.

# Exception Levels

You never execute user programs at the same degree of priviledge as the kernel itself. This is a basic in OS development. If you have a program written by a third party, it may be malicious, so you don't give it too much priviledge to interact with the hardware directly. However, what if some exception occurs in the user program? And what if handling that exception requires you to interact with the hardware directly? In this case when an exception occurs, we go to a higher level of hardware priviledge. 

In Raspberry Pi 3b+'s ARM Cortex-A53 CPU core, this concept is already implemented. Priviledge levels are called "Exception levels". There are four exception levels. EL0, EL1, EL2, EL3. 

- EL0: Used for user programs, lowest hardware priviledge
- EL1: Used for the kernel, higher hardware priviledge
- EL2: Used for virtualization, multiple OS at the same time
- EL3: Used for suuuper low level security stuff, affects the processor itself. Highest priviledge.

The last two will not be relevant to our project. Mainly we will be working with EL0 and EL1. 

Whenever a hardware exception occurs in EL0, either the exception is handled in EL0 itself, or if needed the level is raised and exception is sent to EL1 to be handled.

# Relevant Registers
## `ELR_EL1`
The name stands for "Exception Link Register for EL1". We have learned that whenever an exception occurs, the CPU will change PC to the address of the appropriate exception handler instructions. However once the exception handling instructions conclude their job and handle the exception, the program execution may need to go back to the address where it was originally executing at right before the exception occured. How does the CPU know where to go back to? or where to Return to after an exception handling?

The CPU stores the original PC to return to in the `ELR_EL1` register. However, this happens only if the exception is being handled in EL1. For instance, if an exception occured in EL1, and hardware decided to raise priviledge level to EL2 in order to handle it, now a different register called `ELR_EL2` will be used to store the return address into.

## `SPSR_EL1`
Stands for "Saved Program Status Register for EL1". The CPU has many other registers other than PC which work as memory to hold important information about the current execution. We will discuss them more later, but it includes something called "flags", "Interrupt masks", "program status", etc. When the CPU jumps to exception handler, it also must save this important information in case the exception handler modifies any of it. That is what this register holds. This register holds the PSTATE (program state) information of the program that was being executed right before the jump to exception handler occured. It also holds information about whether if on return, the CPU must stay in same EL or drop lower to EL1.

There are also `SPSR_EL2`, `SPSR_EL3`... for when the exeption is taken to other ELs. However right now we will only see exceptions being taken to EL1.

## `CurrentEL`
Less of an actual piece of memory, but more of a user convenience. Whenever we wish to check what EL we are currently in, we can perform a read on the `CurrentEL` register. Which will return one of the following values: `0b0000` for EL0, `0b0100` for EL1, `0b1000` for EL2, and `0b1100` for EL3. You may have realized to get the CurrentEL as integer, you can simply do `(CurrentEL >> 2) & 0b11`

## `ELR_EL2`
Same as the EL1 counterpart. Holds the address to return to upon `ERET` instruction if currently in EL2.

## `SPSR_EL2`
Same as the EL1 counterpart. When we ERET while in EL2, this register is checked to see which state must be 'restored' upon returning to the address in `ELR_EL2`

## `SP_EL0`
This is the stack pointer register for EL0 mode. Higher ELs can also access it. User programs typically use this stack pointer. in EL0 mode, `sp` register directly references to this register.

## `SP_EL1`
Same as earlier but for EL1 and higher modes. EL0 cannot access this. Mainly for the stack pointer of the kernel programs executions.

## `VBAR_EL1`
Stands for Virtual Address Base Register for EL1. Earlier we mentioned that we need to tell the CPU through a register, which address to jump to when an exception occurs. This register holds the address.

But wait.. Only one register? Aren't there many kinds of exceptions possible as earlier discussed? Is there a different register for every single exception type handler?

No. Firstly, there can be four kinds of exceptions:w
- Synchronous exception (SYNC): caused directly because of execution of some instruction that warranted an exception occurance.
- Interrupt Request (IRQ): When some hardware event occurs, like a network card receives some data, or UART receives new bytes, the hardware generates an exception to get the CPU's attention to let it know that the hardware event has occured.
- Fast Interrupt Request (FIQ): Similar to IRQ, but for wayyy more higher priority events. They are categorized into this type because they may need special more higher priority attention as soon as possible.
- System Error (SError): For hardware failures that occur at some random point, independant of instruction execution progress. E.g. an instruction asked for some data, it gets the acknowledgement, execution moves on. But later on during the actual data transfer the hardware has some failure.

There is another way to differentiate exceptions based on the mode of execution right before exception occured:
- EL1t (t stands for thread): If before exception, program was executing in EL1 mode, and using the SP_EL0 register as the stack pointer.
- EL1h (h stands for handler): If before exception, program was executing in EL1 mode, and usnig the SP_EL1 regsiter as the stack pointer.
- EL0 64 bit: Before exception we were executing in EL0, in 64 bit mode.
- EL0 32 bit: Before exception we were executing in EL0, in 32 bit mode.

Yes, as you may have noticed you can choose which of the two registers are used as stack pointer in EL1. We will discuss that later. For now just know that it can be done.
Secondly, yes, EL0 could be running in both 32bit and 64bit modes. The entire processor can switch between the two modes. But you can also selectively choose for only EL0 to run in 32bit mode.

Now, for every type of Exception, there can be four sources of where it came from. (EL1t, EL1h, EL0 32 and 64.) So in total, there could be 16 possible combination of cases which need separate handlers. Does this mean you need 16 registers for the address of each register? The answer is no. The CPU will expect you to keep all the 16 handlers next to each other in memory, with exactly 128 bytes of difference between each handler's address. It has to be in the following order: firstly for EL1t: SYNC handler, then 128 bytes later IRQ handler for EL1t, then FIQ, then SError. Then the same order for EL1h, then EL0 64 bit, and then 32 bit. 

So the CPU actually expects the handlers to be stored like this in memory:
let's say the handler for EL1t, SYNC exception is stored at 0x00, then rest will be at addresses:


| Exception Origin | SYNC | IRQ | FIQ | SError |
|--------|---|---|---|---|
| EL1t | 0x000 | 0x080 | 0x100 | 0x180 |
| EL1h | 0x200 | 0x280 | 0x300 | 0x380 |
| EL0 (64 bit) | 0x400 | 0x480 | 0x500 | 0x580 |
| EL0 (32 bit) | 0x600 | 0x680 | 0x700 | 0x780 |

And then that's the neat part, you only need to tell CPU the address where this table begins. That is, address of EL1t SYNC Exception handler. The rest it already knows will be at offsets of 128 bytes.

Now, you may have noticed that this means for each exception handler you only get like 128 bytes, and if one instruction is of 4 bytes, you can only write 32 instructions for each handler, including `ERET`. This is not necessary. You can kind of cheat this limitation by simply using a `BL` instruction or something similar, to simply jump to some other location in memory where the actual handler code exists. This is typically the standard way in most OS. 

## `HCR_EL2`
Stands for Hypervisor Configuration Register. EL2 is often called hypervisor mode. This register is suffixed with EL2 because it can only be modified in EL2 or higher priviledge mode. This register acts as a rulebook for EL1. What mode EL1 executes in (32 or 64 bit), whether EL1 exceptions should directly be taken to EL2 or not, interrupts setup, memory translation, etc are configured here. When the system boots the state of this register is random (UNKNOWN).

## `ESR_EL1`
Stands for Exception Syndrome Register for EL1. Holds more detailed information about the identity of the exception. Only works if exception was of SError type. Otherwise it is useless.

## `FAR_EL1`
Stands for Fault Address Register for EL1. If and only if the exception was of type SError, and it involved some sort of memory failure, this register will hold that exact virtual memory address that caused the crash. Whether or not the SError was related to memory can be confirmed through information held in `ESR_EL1`. This register is completely useless in other scenarios.


Well, that wraps it up!!! That was a lot of information to memorize, it will take time to truly get to know all of these registers personally.


# Booting to EL1

When you start your OS. It actually starts in EL2 for the Raspberry Pi 3 CPU. Since we want our OS to work in EL1. We will need to switch to EL1 in boot process in `entry.s`. Sadly there is no actual direct way to just switch over to a different EL. But the correct way to do it is much more clever.

You already know if we are in EL1, and an exception occcurs, and the hardware decides that it needs to be taken to EL2, then the CPU raises priviledge to EL2, and then exception is handled there. Wherein at the end `ERET` instructionn is executed which causes the CPU to go back to EL1.

But the trick is you don't actually need to be handling some exception for `ERET` to send you back to EL1! The CPU is just a machine, it does not know what was going on. All it does is if you execute `ERET` in EL2, it will check `ELR_EL2` to know which address to go to, and it will simply go there. And it will check `SPSR_EL2` to know what mode to set the execution to once it goes to that address. You don't need to be handling an exception! As long as you set `ELR_EL2` and `SPSR_EL2` to some valid values, you can use `ERET` whenever you want in EL2!

*So...* we *could* simply set `SPSR_EL2` in such a way that the CPU thinks that upon `ERET` it must change EL to EL1.

First know the general structure of a SPSR register:

```arm
Bits [4:0]  = M field (mode)
Bit  6      = F (FIQ mask)
Bit  7      = I (IRQ mask)
Bit  8      = A (SError mask)
Bit  9      = D (Debug exception)
Bits [31:27] = NZCVQ flags
Other bits = Reserve or not relevant right now
```

- Bits [4:0] are:
  - 0b00000: we're supposed to enter EL0 mode after `ERET`  
  - 0b00100: EL1t after `ERET`
  - 0b00101: EL1h
  - 0b01000: EL2t
  - 0b01001: EL2h (all five being AArch64 mode)
- Bits 6, 7, 8 are for turning off exception handling for FIQ, IRQ and SError type exceptions (set bit to 1 to ignore them). SYNC type exceptions do not have an option to be ignored because the CPU cannot continue execution without them handled. 
- Bit 9 handles Debug exceptions. They are usually SYNC exceptions, but unlike failure they're usually for things like breakpointns, debugging, etc.
- Bits 31 to 27 are just the flags that the CPU sets during instruction execution. They are just saved here upon exception because upon ERET the following instructions may rely on flags set by previous instructions that were executing before exception occured.

Now, in order to switch to EL1 mode. All we need to do is set Bits 4:0 to 64 bit EL1h mode. (Not EL1t because we're switching for our kernel, and kernel typically uses SP_EL1). Set appropriate values for the interrupt masks and endianness, and then simply execute `ERET`!

We can do this during our entry assembly.

Now, in our entry.S instead of directly doing `bl _rust_main` we do:

```arm
// EL stack pointer
ldr     x0, =_stack_top
msr     SP_EL1, x0

mov     x0, #(1 << 31) // bit 31 selects Aarch32/64
msr     HCR_EL2, x0 // HCR_EL2 is like "a rulebook that EL2 writes for EL1"

// set SPSR_EL2 to EL1h, basically once we ERET, we must switch to EL1h mode
mov     x0, #(0b00101)  // EL1h 
orr     x0, x0, #(0b1111 << 6) // mask all exceptions (D, A, I, F)
msr     SPSR_EL2, x0

// finally set the ERET PC
adr     x0, _rust_main
msr     ELR_EL2, x0

eret // insane
```

Notice that we also set bit 31 in `HCR_EL2` register to 1. This enables 64 bit mode for EL1. Other bits are irrelevant to us right now.

Now let's create a utility function that can read `CurrentEL` register so we can confirm our code is working:

```rs
use core::arch::asm;
pub fn get_current_el() -> u64 {
    let el: u64;
    unsafe { 
        asm!(
            "mrs {0}, CurrentEL",
            "lsr {0}, {0}, #2",
            out(reg) el,
        ); 
    }
    el
}
```

Now just include this function in your main.rs and test it out!

```rs
println!("Current EL is: EL{}", get_current_el()).unwrap();
````

You get output:
```
Current EL is: EL1

```
That means our EL is successfully EL1!

We can now move to exception handling.

# Setting up and loading exception table
