# below text is currently under progress
# User Processes

Your operating system is not for running by itself. Ultimately of course the goal would be for it to run processes for the user, and for the user to be able to run whatever process they want on it. Which, of course, our OS will need to achieve in a safe manner. Without letting the user processes access the hardware through any CPU or MMIO registers which could be dangerous. Our ultimate goal for our operating system would be to facilitate a safe usage of hardware by the processes, in a asynchronous manner. That is, in a manner where multiple processes can run at the same time.

# But what is a Process? 

Your computer or phone may have many different apps or programs on it. Normally something like your calculator app is not running on your device. It is usually stored on your storage/disk. However, when you wish to open the calculator app, somehow your operating system turns that bit of program on your disk into a running functioning execution of code. To put it plainly, the operating system reads the instructions and data of the app, and writes it to the memory (RAM). And then it sets `PC` to the first instruction meant to be executed from this newly loaded set of instructions in memory. Execution then continues and your app appears to run.

However, your app/program is still on your disk. It's not as if when the program is loaded to memory, it is erased from the disk. If that were the case, then every time a power outage happened in the middle of execution, your app would disappear from your device (since RAM memory is lost upon power loss). One could say that the program and data of the application on your disk is just there for your OS to load the correct instructions into memory. However, once those instructions are loaded into memory and they began executing, what do you call them? What do you call this bit of chunk in your memory that is executing instructions of your application? It needs a name to. And we call it a "Process".

Simply put, a process is running instance in memory of any program. At a bare minimum the way of creating a process is pretty similar to loading our `kernel8.img`. For a process, you go through its data/files on the disk to figure out what its image in memory will look like. And then you simply place it in memory at an appropriate memory location. Then you set a stack pointer for it, and simply set `PC` to the entry point of that process, i.e. the first instruction meant to execute for this new set of instructions. However, it differs from our `kernel8.img` in the sense that our operating system will keep track of processes and give them an identity. They will have a unique identification number, a name, information about who created them, their memory addresses, etc. Our operating system will then control how they interact with the hardware and decide which process gets to run at the moment. We will also manage new processes running and terminating. 

# Loading a simple process

A process will have many different parts to it. Also, deciding where to place the process and how to prevent it from accessing MMIO addresses will be a complicated procedure covered under Memory Management. Memory management is something which we will implement after we have some basic processes running and executing together on the CPU. For now, we will first create a process image on our host system itself, and just include it in our `kernel8.img`. Since it is included in `kernel8.img`, it will be loaded alongside to memory when RPi loads `kernel8.img` to memory. From their our kernel will copy that process image from `kernel8.img` section, over to where it is actually supposed to be. 

How do we decide where to place it? First of all, the user processes will be compiled in a manner extremely similar to how our kernel is compiled right now. We will create a separate cargo project named "user" which will also target baremetal AArch64 Raspberry Pi platform. And exactly like how our kernel project has a linking script which tells our compiler where the kernel will be loaded in memory (which is at 0x80000), we will also have a linker script in the `user` project, where instead of 0x80000, we will decide where we will place our process in memory. We will then write the linker script to start at our chosen address (any free address in memory can work).

Then again, the same way we first compile our OS project to an ELF, then a kernel8.img is dumped out of that elf. We will also dump an image from the compiled ELF produced in the user project. And then this image will be loaded by our running OS kernel into memory at the correct intended location. 

Then for starting the execution of the loaded program. We can simply check the original compiled ELF to find out the entry point address for the image. Then simply jumping to that address in memory.

However, recall that we actually are supposed to have user programs to run in EL0. And also that any running program executing in EL0 is actually going to use the `SP_EL0` register to find the stack pointer. So we will need to setup that as well. It is a very similar process to how we dropped from EL2 to EL1 into our kernel. As we will see.

# User project

Firstly lets start off by creating said user project. with the following in our project directory.

```cargo new user```

Then `cargo.toml` can be something like:

```toml
[package]
name = "user"
version = "0.1.0"
edition = "2024"
authors = ["ZackyGameDev <zaacky2456@gmail.com>"]
```

Next up, in this project we will have two things we need to write.

1. **The different user programs.** Of course we will train our OS to handle multiple processes running on it at the same time. And for achieving that, we are going to need different user programs that will be loaded into the memory at separate places at the same time.

2. **a common standard library.** Right now our user programs are going to be able to access MMIO memory address freely. However, in the future we are going to limit it to prevent user programs from doing that. But then how will a user program print something to the output if it cannot access UART registers through MMIO? The answer is that when the user program requires some operation which needs to access MMIO or any other priviledged/dangerous feature, it will need to send a request to kernel. These request are handled through "syscalls". And all of these syscalls will be accessible to the user program in common module that is generally called the standard library.

The standard library also defines higher level featuers which aren't syscalls by themselves, but may rely heavily on syscalls to function. Or provide other structures and methods that a common user might need (for e.g. `String` struct in `std` in Rust).

(to continue)


(
