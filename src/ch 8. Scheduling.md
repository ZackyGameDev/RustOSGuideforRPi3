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


