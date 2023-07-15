---
title: "RISC-V Emulator part 5: Implementing RVA"
date: 2023-07-15T15:57:07+02:00
tags: ["rv64gc_emu", "RISC-V", "RISC-V Emulator"]
gh_link: rv64gc-emu
previous_post: /posts/rv64emu-part-4
---

This will be a brief post about some design details specific to RVA:

# Avoiding implementation repetition

A lot of instruct instruction do the same thing in 32-bit and 64-bit RVA instructions. To avoid code dumplication, we can template the functions. For example, we can `amoaddw` and `amoaddd` this way:

```cpp
template <typename T>
{
    Cpu::reg_name rs1 = decoder.rs1();
    Cpu::reg_name rs2 = decoder.rs2();
    Cpu::reg_name rd = decoder.rd();

    uint64_t addres = cpu.regs[rs1];

    T val1 = cpu->bus.load(addres, 32);
    T val2 = cpu.regs[rs2];

    T result = val1 + val2;

    cpu->bus.store(addres, result, 32);

    cpu.regs[rd] = SIGNEXTEND_CAST(val1, T);
}
```

Then, during decoding, we can do:

```cpp
case AtomicType::ADD:
        atomic::amoadd<int32_t>(cpu, decoder);
        break;
...

case AtomicType::ADD:
        atomic::amoaddw<int64_t>(cpu, decoder);
        break;
```

# Instructiong aligment

When dealing with RVA, we need to make sure that the destination addresses are naturally aligned. This means that the address of the instruction is divisible by 4 for 32-bit instructions and by 8 on 64-bit ones. If it is not, we need to throw an exception. We can do this by adding the following to functions that decode RVA's `funct5`:

```cpp
// For 32-bit
if (cpu.regs[decoder.rs1()] % 4 != 0)
{
    perror("Missaligned instruction");
    exit(1);
}

// For 64-bit
if (cpu.regs[decoder.rs1()] % 8 != 0)
{
    perror("Missaligned instruction");
    exit(1);
}
```

# Reservations

RVA has a concept of reservations. This means that if we do a `lr.w` instruction, we will reserve the address of the instruction. If we do a `sc.w` instruction, we will check if the address is reserved. If it is, we will store the value. If it is not, we will throw an exception. We can implement this by adding a `std::set<uint64_t> reservations;` inside of our `Cpu` class. Then, we can implement `lr.w` and `lr.d` in the following way:

```cpp
template <typename T>
void atomic::lr(Cpu& cpu, Decoder decoder)
{
    Cpu::reg_name rs1 = decoder.rs1();
    Cpu::reg_name rd = decoder.rd();

    uint64_t addres = cpu.regs[rs1];

    T val1 = cpu->bus.load(addres, 32);
    cpu.regs[rd] = SIGNEXTEND_CAST(val1, T);

    cpu.reservations.insert(addres);
}
```

And `sc.w` and `sc.d` in the following way:

```cpp
template <typename T>
void atomic::scw(Cpu& cpu, Decoder decoder)
{
    Cpu::reg_name rs1 = decoder.rs1();
    Cpu::reg_name rs2 = decoder.rs2();
    Cpu::reg_name rd = decoder.rd();

    uint64_t addres = cpu.regs[rs1];

    T val2 = cpu.regs[rs2];

    if (cpu.reservations.contains(addres))
    {
        cpu.reservations.erase(addres);

        cpu->bus.store(addres, val2, 32);

        cpu.regs[rd] = 0;
    }
    else
    {
        cpu.regs[rd] = 1;
    }
}
```

If you need further help with implementation details, you can check out [this](https://github.com/bane9/rv64gc-emu/blob/main/source/instructions/atomictype.cpp).

In the next post, we will implement a trap system for handling exceptions and interrupts.