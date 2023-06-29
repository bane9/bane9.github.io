---
layout: post
title:  "Writing a RISC-V Emulator that can boot Linux"
date:   2023-06-25 16:00:00 +0200
project: rv64gc_emu
project-link: rv64gc-emu
next-post: "RISC-V Emulator part 1: Getting started"
---

# Introduction

For the past several months I have been working on a RISC-V emulator, mainly inspired by [mini-rv32ima](https://github.com/cnlohr/mini-rv32ima). The goal of my project was to create such an emulator myself, but with some key differences: a 64-bit core instead of a 32-bit one, implemnting the FPU, compressed instructions, and a MMU. The end goal was to be able to boot Linux on it as well as have a graphical output that will be used maily to play doom.

I'm happy to say, all of it was achieved. Now I'm writing this blog to help people to do the same thing. The end goal is to document everything needed to be done to achieve this:

<video controls="controls" src="https://user-images.githubusercontent.com/29211832/247784659-0b25afea-3062-4471-b5e4-7e286e6425f1.mp4" allowfullscreen></video>

This will entail a series of blogposts that will eventually contain the following stuff (subject to change):

- Explanation of what RISC-V is and how it works
- Implementation of basic emulator concepts (memory, registers, etc)
- Implementation of the core with the following extensions:
  - RV64I
  - RV64M
  - RV64A
  - RV64F
  - RV64D
  - RV64C
  - RV64Zicsr
  - RV64Zifencei
  - RV64Privileged
- Implementation of the MMU along with a basic TLB
- Implementation of peripherals needed for booting Linux
- "Baremetal" Doom port
- Booting Linux and Buildroot configuration that was made to work with the emulator
- Implementation of a framebuffer device
- Getting DOOM to work under Linux

What this series is not/will **not** be:
- A comprehensive guide on how to write an emulator. There are several different ways of doing it (e.g. the naive method, cached emulation, JIT, etc).
- A guide on how to write a high performance emulator. The point
of this project is to primarily be an educational tool, and there are other projects that are much better suited for this purpose. For RISC-V, the ones I reccomend checking out if performance is of concern is [QEMU](https://www.qemu.org/) and [RVVM](https://github.com/LekKit/RVVM)
- A step by step tutorial. This will be more of a documentation of what I did, and I will try to explain the concepts as I go along. This series is meant as an overall guide, and requires some pre-existing knowledge with C++ and computer architecture.

With that out of the way, lets dive in!

# Basic computer architecture

Before we dive into the details of RISC-V, we need to cover some basic computer architecture concepts. If you are already familiar with these concepts, feel free to skip this section.

## Registers

Registers are the fastet type of memory a CPU core can access. They are usually used to store temporary values that are used in calculations. The number of registers a CPU has is usually fixed, and is usually a power of 2. For example, the x86-64 architecture has 16 general purpose registers, while the RISC-V architecture has 32 general purpose registers.

## System memory (aka RAM)

System memory is the main memory of the computer. It is used to store the program that is currently running, as well as the data that is being used by the program. It is usually much slower to access than registers, but it is much larger. The size of the system memory is usually measured in bytes, and is usually a power of 2. For example, the x86-64 architecture has a 64-bit address space, which means it can address 2^64 bytes of memory, or 16 exabytes. The RISC-V architecture has a 64-bit address space as well.

## Instructions

Instructions are the basic building blocks of a program. They are the smallest unit of work that a CPU can do. Instructions are usually encoded as a sequence of bytes, and are stored in the system memory. The CPU fetches instructions from the system memory, decodes them, and executes them. The instructions that are executed are usually stored in the registers.

## Interrupts

Interrupts are a way for the CPU to stop what it is currently doing, and do something else. Interrupts are usually used to handle events that are not related to the current program. For example, a keyboard interrupt is used to handle a keypress, and a timer interrupt is used to handle a timer. Interrupts are usually handled by the operating system, and are usually used to switch between programs.

## System calls

System calls are a way for the program to ask the operating system to do something for it. For example, a program can ask the operating system to open a file for it, or to write to a file for it. System calls are usually handled by the operating system, and are usually used to switch between programs.

## System Bus

The system bus is a way for the CPU to communicate with the system memory. It is usually a set of wires that connect the CPU to the system memory. The system bus is usually divided into two parts: the address bus and the data bus. The address bus is used to tell the system memory which address the CPU wants to access. The data bus is used to transfer data between the CPU and the system memory.

## Instruction Set Architecture (ISA)

An ISA is a specification of the instructions that a CPU can execute. It is usually divided into two parts: the instruction set and the instruction encoding. The instruction set is a list of instructions that a CPU can execute. The instruction encoding is a way to encode instructions as a sequence of bytes. The instruction set is usually divided into two parts: the opcode and the operands. The opcode is a number that tells the CPU which instruction to execute. The operands are the values that the instruction uses. The instruction encoding is usually divided into two parts: the opcode and the operands. The opcode is a number that tells the CPU which instruction to execute. The operands are the values that the instruction uses.

## RISC-V Extensions

RISC-V is a modular ISA, which means along with the base instruction set (aka RVI), the CPU can have an additional set of supported instructions. The ones we are mainly interested in this series are:

- RVI: Base integer instruction set that also contains load/store and jump instructions
- RVM: Integer multiplication and division instructions
- RVA: Atomic memory access instructions
- RVF: Single precision floating point instructions
- RVD: Double precision floating point instructions
- RVC: Compressed instructions
- RVZicsr: Control and status register instructions
- RVZifencei: Instruction cache management instructions
- RVS: Supervisor mode privledge level and associated instructions
- RVU: User mode privledge level and associated instructions


# Architecture of the system we are implementing

![RV64GC Emu System Architecture]({{ "/post_assets/system_arch.png" | relative_url }}){: style=" display: block; margin-left: auto;margin-right: auto;"}

The system we are implementing is a RISC-V RV64GC system. This means that it has the following features:

- RISC-V CPU Core: The CPU core itself.
- MMU: The memory management unit. It's specifics will be covered later down the line.
- System Bus: Common communication bus between the CPU and the system memory and peripherals.
- System Memory: The main memory of the system. It is used to store the program that is currently running, as well as the data that is being used by the program.
- UART: A serial communication interface. It is used to communicate with the host machine.
- PLIC: A programmable interrupt controller. It is used to handle interrupts.
- CLINT: A core-local interrupt controller. It is used to handle timer interrupts.
- VIRTIO MMIO: A virtual I/O device. It is used to communicate with the host machine.
- Framebuffer device: It is used to display graphics on the screen.

---

This has been an introduction to the series. In the next part, we will set up our development environment, and start implementing the CPU core. Join me in part 1 (starting here as 0, obiously)!