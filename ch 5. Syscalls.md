# below text is still under work

# User space isolation

So far what we've done is that we have created a separate project directory which acts as the place where all the user's programs and standard library will be written. Basically all the stuff that is supposed to run ON our operating system. However why did we create a separate project for it? Why couldn't we just keep growing our original operating system repository? Well, there's mainly two reasons.

the first reason is that user program is not just supposed to be the operating system related processes that run for the user like the shell, window manager, desktop environment, etc. The user programs can also be third party codes or applications that are written by people who are not associated with us or our operating system's development. Those third parties should have a way to write applications that can run on our operating system without having to know how the OS itself handles the hardware. So this produces sort of a design philosophy based reason to separate the user program space including the user standard library, from the kernel project itself. 

When a third party includes or imports our standard library that we wrote, they will also work in a project environment completely disconnected from our OS kernel source code. And so we make the entire standard library as a standard rust library package project. 

The second reason is because of security. We know that by letting third parties create their own programs for our operating system, we allow them to freely introduce new amazing features or programs that do cool things on our operating system. However there can be malicious third parties which for whatever reason, simply wish to cause chaose on the hardware which is running our operating system. They might try to directly access RAM to violate the memory space used by other running programs. They might try to access the disk/microSD in order to corrupt files. They might try to access the MMIO addresses in order to control the rest of the hardware maliciously.

This is the entire reason we drop to EL0 before letting a user program execute. This is because from kernel which runs in EL1, we can manually tell the hardware to block/moderate EL0's direct access to the hardware. Completely blocking the program running in EL0 from even being able to see any other process's memory in RAM, or from even being able to access the MMIO or disk.

Thus, user programs should never expect to be able to access the hardware directly like the kernel can. So we also separate our rust project for our user standard library to a different cargo project. If user programs were compiled within `kernel8.img` then it would be more complicated to separate them and isolate them for running them in EL0.

# Syscalls

However, if the user programs cannot even expect any access to the hardware, how would they even interact with the machine and do whatever they are designed to do? How could they print output to the user without access to a display/UART? How could they read input from the user without reading data from the keyboard/io/UART_RX? 

To understand lets first define these tasks which require direct sensitive accessing of the hardware as "priviledged jobs" and let the accessing type itself be called "priviledged action". 

Now, to put it plainly, most operating system standards actually expect the user program to send a request to the kernel for the priviledged job. From there, the kernel identifies what sort of priviledged job the user program wishes to perform. And then the *kernel performs* the priviledged task for the user in a safe controlled manner. The user program then receives the result of the privlidged action from the kernel. Usually in the form of return value in a register or at a place in memory that is accessible to the particular user program.

This "request" to the kernel from the user program is called a "syscall". In the following text you will learn how to make your user program send these "syscalls" to the kernel and how the kernel can indentify them, handle them, and then send responses back to the user program.

# Trapping to the kernel

In standard lingo you will often hear the word "trap". Usually when textbooks or lectures talk about the user program sending a request to the kernel, They call it as the user program "trapping into the kernel". What they mean by that is that like how we drop from EL1 to EL0 using the `ERET` method, we can also raise the level from EL0 to EL1. And just like how when dropping to EL0 we start off in the user program, i.e. the moment you drop to EL0 we simultanously switch to the user program's instructions in memorry. We also start in the kernel when we raise the exception level to EL1. This is usually achieved by a dedicated instruction that the EL0 program can execute which causes this to happen. This entire process of using the special instruction in a low priviledge level to manually send the priviledge to a higher level while handing over the control to the kernel is generally called as "trapping into the kernel".

In ARM architecture, the special instruction that lets us send execution to kernel in EL1 is `svc`. We will learn about this instruction more. But it is short for "Supervisor Control". The word "supervisor" is an old classic name of the "kernel". It accepts one immediate argument, so the full way of writing it wuld be somethign like `svc #imm` where `imm` can be any number from 0 to 65535 (`0xFFFF`)

When `svc` instruction is executed, the behavior that is triggered in the hardware can be described as following:
- A hardware exception occurs.
- The exception is taken to EL1. 
- The immediate argument to the `svc` instruction is encoded in the `ESR_EL1` register.
- The type of exception is "Synchronous exception". So the execution jumps to the appropriate entry in `VBAR_EL1` exception table, for `EL0 64 bit` source (since currently our user program executes in `EL0 64 bit` mode).

Thus the kernel's exception handler executes upon the `svc` instruction. The `svc` instruction does not touch any GPR values. So the user program has the option to store relevant information in the general purpose registers before using `svc`. Wherein the kernel's exception handler can identify the values written in the registers. This is how the communication from the user program to kernel occurs for the syscalls.

In linux kernel, different priviledged actions like "write to disk" or "read from disk" or "create new process" are all given a unique identification number. Then before `svc` the user program stores the ID of the requested priviledged action in the `x8` register, which the kernel reads and indentifies.

(to be continued...)