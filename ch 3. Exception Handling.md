# below document is currently under progress

# Hardware Exceptinos

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

(to be continued)
