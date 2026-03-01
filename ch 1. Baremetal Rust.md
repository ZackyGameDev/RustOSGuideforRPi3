# [under work]

# Baremetal Rust

Now that you know exactly how code is actually run and what it looks like after compilation, we have to actually make it run. We are trying to write an Operating System in rust. So of course we need to figure out how to make program written in rust run directly on our RPi without any underlying operating systems.

# Project init
###### NOTE: This section is for Linux (where you'll write your rust code and compile it for the Pi). If you're using a different OS some things may or may not be different. Feel free to take help from LLMs or other online resources for setting up your baremetal rust project with aarch64-unknown-none as your target.

Let's first initialize our bare bones project. 
But before we do that we need to run:

```bash
rustup target add aarch64-unknown-none
```

This will install and add whatever Rust compiler needs to be able to compile code to instructions that can run on "AArch64" architecture (basically RPi3's CPU architecture). The "unknown" and "none" refer to the "vendor" and "underlying operating system" of your target respectively.

Now you can initialize your project with `cargo`. (comes with rust)

```bash
cargo new atos --bin
cd atos
```

Now in your project folder there must be a file named "Cargo.toml". You can add some basic information about your project there. For me it was:

```toml
[package]
name = "at-os"
version = "0.1.0"
authors = ["ZackyGameDev <zaacky2456@gmail.com>"]
```

Next, we have to tell our project what our rust code is meant to be compiled for. Create a file `.cargo/config.toml` which will have information about our compiling settings. and put the following in it:

```toml
[build]
target = "aarch64-unknown-none"
```

Optionally, if you're using vscode then tell the linter/syntax checker what you're targetting by putting the following in `.vscode/settings.json`:

```json
{
    "rust-analyzer.check.allTargets": false,
    "rust-analyzer.cargo.target": "aarch64-unknown-none"
}
```

Now there's still some more we have to setup before we can actually build it, but before that we have to understand some more things and write a little bit of code.

# Lack of Underlying OS

In rust, and in pretty much any programming language actually, there are many features that rely on Operating Systems beneath them, for them to work. For example in C, when you want to open a file, the instructions from your compiled C code itself don't really read the disk directly trying to decipher the data on the disk to find your file and start writing to it. Instead, your code will ask the Operating System beneath to do it for you, and just give you something like a pointer to the file. This is usually so programs written don't need to access the memory directly (becaues if they're evil or terrible they might try to mess up with parts of memory you wouldn't want them to mess up). 

If you've read up on this, you'd know this way of asking the operating system for something is called "System Calls". Where essentially the programming language (in user space) is relying on the kernel (in kernel space) to provide low level interactions through system calls. 
(read about it at [OSTEP: Ch6](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf))

Well what about when you're running your code directly on the computer without any operating system beneath? What if there's no "kernel" to do system calls to? It's simple. Those features become unusable and pretty much useless. And in rust, "those features" includes the entire Rust Standard Library called `std`.

# Lack of `crt0`

Rust code, normally on windows or linux just running normally on the same machine, doesn't run directly. What happens under the hood is that first, something called the "C Runtime Library" is run. What it does is basically setup the environment and parameters of the hardware, in preparation for Rust code to run. It first does something called "setting up the stack" and "initializing .data" or "zeroing the .bss" (we'll learn what it is later). And then it literally calls the function named "main" in your rust code. (Which is why the first lines of code to run are written in a function called "main()", because that's the name `crt0` calls).

As you may have guessed, when making our rust on baremetal, there is no such thing. The labour of "setting up stack" or preparing environment for rust is also going to be need to be done by you manually, before your rust code even starts. But wait... if we have to write some code that does some stuff before Rust code can run... that means that preparation code can't be written in Rust... The truth is unavoidably, we'll have to write this preparatory code in assembly. Because it's pretty much the only language that can run immediately without any preparation (because you're literally writing the structure of the machine code directly). Although you can try to minimize it as much as possible, in this entire project you'll have to unavoidably write some bits in assembly 

# Assembly entry point and preparing for Rust main

Time to finally write those first ever lines of code that will run at `0x80000` (I hope you know the relevance of this address by now).

You can put your file anywhere, for me I put it at root of the project, `./entry.S`. This is what it looks like:

```asm
.section ".text.boot"
.global _start

_start:
    // set stack pointer
    ldr   x0, =_stack_top
    mov   sp, x0

    // jump to rust
    bl   main

1:
    wfe
    b 1b
```

Let's understand this line by line. The first line defines the section this code will go to (discussed under linking further below). Then `.global _start` just means "this symbol called `_start` should be accessible globally, to all files and codes". This is just for the linking discussed further below.

Next, the first lines of code (labeled under `_start`). All we're doing is loading up an address `_stack_top` in the `x0` register of the chip, and then copying that value to the `sp` register from `x0` register. We load it into `x0` first because there's no instruction to copy an address directly to `sp` register. (If this seems alien, check out ch a. Appendix on ARM registers.

And then finally we do `bl main` which literally just means "call the code under the label `main`". Where is main you ask? Well.. that's the main function in our rust code of course! So what this is essentially doing is calling the rust main function after setting the `sp` register to some value.

You might've noticed we just pulled this variable called `_stack_top` and just assumed it has the correct address to set the stack to. But where was this defined? Where did it come from? Well again, it is defined in the linking process. Assembly doesn't really have variables. It's not like there's some variable defined on the stack or heap which the assembly accesses like other programming langauges. If it worked like that we would've written the instructions to access data from the heap and use it. This is actually something called a "symbol". kind of like inline macros from C. The linker while producing the final output file will see this symbol, and try to find where that symbol is defined. And then everywhere in code it will replace the symbol with its correct defined value.

So where is the `_stack_top` symbol defined? We'll define it during the linking process further below. 

In any case, as it turns out setting `sp` register to the correct value happens to be pretty much all you need to do to call your `main` funciton in rust and expect it to work correctly. So that's what we do. After rust main is called, we'll probably write our rust code in a way 

### [ ... todo ]

## LInking????

In chapter 0 I mentioned that you will be able to tell RPi, which memory addresses to put what code or what data in when booting it. The way it works is the following: When you compile your code (by using the build commands we will talk about in a bit), all the files in your code are first compiled, i.e. translated to machine code. Then something called the "Linker" combines all those machine codes into a single file. Which we give to our raspberry pi; to copy to the memory after booting. Now RPi will always just copy the entire file exactly one to one to it's memory. so data at address 0x80000 in the file will be copied to the same address 0x80000 in the memory. So to decide what data goes to which memory, you have to put it at the correct place in the compiled file itself. 

Since it is the Linker who combines all the code into a single file, it also provides us the option to let it know which code should go to which addresses in the final compiled file. This is done through something called a "Linker script". To be exact, we're going to give a "label" to our code which we want to run immediately after RPi boots. And tell the linker to put all the code with that label at the address "0x80000". (Because as discussed last chapter, that is the hard fixed address that RPi starts executing instructions from after booting).





https://github.com/ZackyGameDev/AtOS/tree/81fcfb3cbdbd0fb0add78b5c89ff8e5bad70d260
