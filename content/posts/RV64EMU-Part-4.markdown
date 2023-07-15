---
title: "RISC-V Emulator part 4: Compiling and running RISC-V binaries"
date: 2023-07-15T15:21:34+02:00
tags: ["rv64gc_emu", "RISC-V", "RISC-V Emulator"]
gh_link: rv64gc-emu
previous_post: /posts/rv64emu-part-3
---

To test out our emulator, we need to write some assembly and compile it. To do that, we first need to get a compiler toolchain. While we can use pre-compiled ones available on brew/apt, I had bad experience with them being improperly configured or had old builds with certain bugs present. Instead, we will build our own toolchain. 

# riscv-gnu-toolchain

To build and install the toolchain, we will use [riscv-gnu-toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain). First, clone the repository:

```bash
git clone https://github.com/riscv/riscv-gnu-toolchain
```

Then, we need to install some dependencies:

**Linux:**
```bash
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev
```

**macOS:**
```bash
brew install python3 gawk gnu-sed gmp mpfr libmpc isl zlib expat texinfo flock
```

Now, lets configure it. From the root of the repository, run:

```bash
./configure --prefix=/opt/riscv --with-cmodel=medany
```

When we invoke the build, it will install it inside `/opt/riscv` directory. We also set the `cmodel` to be `medany`. This is needed, as it will compile libc in a way where it can be linked in an address space that is larger than 2 GiB. If you recall, were putting all our RAM at 0x80000000 which is 2 GiB, so we need to compile libc in a way that it can be linked in in our environment.

Now, lets build it:

```bash
sudo make -j$(nproc)
```

And lets add it to our path (assuming bash):

```bash
echo 'export PATH=/opt/riscv/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

# Writing and compiling assembly

This won't be a comprehensive guide on writing RISC-V assembly. Let's just write this basic program:

```asm
.section .text
.globl _start

_start:
    # Load constants from memory
    lui t0, %hi(constant1)    # Load upper immediate of constant1 address
    lw t1, %lo(constant1)(t0) # Load constant1 into t1 register

    lui t0, %hi(constant2)    # Load upper immediate of constant2 address
    lw t2, %lo(constant2)(t0) # Load constant2 into t2 register

add_loop:
    add t3, t1, t2            # Add constant1 and constant2, store result in t3 register

    # Infinite loop
    j add_loop

.section .data
.constant1:
    .word 10                  # Define constant1 value
.constant2:
    .word 20                  # Define constant2 value
```

RISC-V assembler also has a concept of pseudo instructions. The above code can be simplified to:

```asm
.section .text
.globl _start

_start:
    # Load constant1 into t1 register
    lw t1, constant1

    # Load constant2 into t2 register
    lw t2, constant2

add_loop:
    add t3, t1, t2            # Add constant1 and constant2, store result in t3 register

    # Infinite loop
    j add_loop

.section .data
constant1:
    .word 10         # Define constant1 value
constant2:
    .word 20         # Define constant2 value
```

Notice that while loading addressed directly into registers is not something supported by the CPU, the assembler will add the necessary instructions needed for that to happen. The list of pseudo instructions can be found [here](https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md#-a-listing-of-standard-risc-v-pseudoinstructions).

And lets compile it as:

```bash
riscv64-unknown-elf-gcc -nostdlib -nostartfiles -ffreestanding -march=rv64im -mabi=lp64 -Ttext=0x80000000 -o add add.s
```

Lets go trough the arguments:
- `-nostdlib` and `-nostartfiles` will tell the compiler to not link in the standard library and startup files. We will provide our own startup code.
- `-ffreestanding` will tell the compiler that we are compiling for a freestanding environment, and that it should not assume that the standard library is present.
- `-march=rv64im` will tell the compiler that we are compiling for RV64IM architecture. This will enable all the instructions we need.
- `-mabi=lp64` will tell the compiler that we are compiling for LP64 ABI. This will make sure that the compiler will use the integer registers.
- `-Ttext=0x80000000` will tell the compiler that the entry point of the program is at 0x80000000.

The output of the compiler will be an `elf` file, which our emulator can't run. We need to convert it to a binary file. We can do that with `objcopy`:

```bash
riscv64-unknown-elf-objcopy -O binary add add.bin
```

We can also use `objdump` to directly disassemble the binary, this will be useful for debugging:

```bash
riscv64-unknown-elf-objdump -d add > add.dump
```

Now, we can run the binary with our emulator:

```bash
./emulator add.bin
```

You can use a debugger like `gdb` to debug the program to step trough instruction execution in your emulator and check if it matches the output of `objdump`.

In the next post, we will go trough high level implementation details of RVA.