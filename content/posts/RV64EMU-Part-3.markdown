---
title: "RISC-V Emulator part 3: Instruction execution"
date: 2023-07-15T14:25:58+02:00
tags: ["rv64gc_emu", "RISC-V", "RISC-V Emulator"]
gh_link: rv64gc-emu
previous_post: /posts/rv64emu-part-2
draft: true
---

In this post, we will start implementing the instruction execution logic. We already fetched the instruction and created a decoder class, now lets tie everything together to ghave RVI fully implemented.

# Instruction execution

Since standardized RISC-V instructions are either 32-bit or 16-bit, we will need to implement two different instruction execution functions. Lets add `execute32` inside our cpu class, where we will switch on the opcode and call the appropriate function:

```cpp
void execute32(Decoder decoder)
{
    switch (static_cast<OpcodeType>(decoder.get_opcode()))
    {
    case OpcodeType::LOAD:
        rvi::load_funct3(*this, decoder);
        break;
    case OpcodeType::I:
        rvi::typei_funct3(*this, decoder);
        break;
    case OpcodeType::S:
        rvi::types_funct3(*this, decoder);
        break;
    case OpcodeType::R:
        rvi::typer_funct3(*this, decoder);
        break;
    case OpcodeType::B:
        rvi::typeb_funct3(*this, decoder);
        break;
    case OpcodeType::I64:
        rvi::typei64_funct3(*this, decoder);
        break;
    case OpcodeType::R64:
        rvi::typer64_funct3(*this, decoder);
        break;
    case OpcodeType::AUIPC:
        rvi::auipc(*this, decoder);
        break;
    case OpcodeType::LUI:
        rvi::lui(*this, decoder);
        break;
    case OpcodeType::JAL:
        rvi::jal(*this, decoder);
        break;
    case OpcodeType::JALR:
        rvi::jalr(*this, decoder);
        break;
    default:
        perror("Unknown instruction");
        exit(1);
        break;
    }
}
```

Lets also add `execute16`:

```cpp
void execute16(Decoder decoder)
{

}
```

Now, lets expand `do_insn()`:

```cpp
bool do_insn()
{
    uint32_t insn = fetch();
    Decoder decoder = Decoder(insn);

    uint32_t insn_len = decoder.get_len();

    if (insn_len == 2)
    {
        execute16(decoder);
    }
    else
    {
        execute32(decoder);
    }

    pc += insn_len;

    return true;
}
```

# Implementing instructions

First, lets create a new file `source/instructions/rvi.cpp`:

```cpp
#include "rvi.hpp" // Creating this file is an excercise for the reader
#include "cpu.hpp"

namespace rvi
{
    void typei_funct3(Cpu& cpu, Decoder& decoder)
    {
        switch (decoder.funct3())
        {
        case IType::ADDI:
            addi(cpu, decoder);
            break;
        case IType::SLLI:
            slli(cpu, decoder);
            break;
        case IType::SLTI:
            slti(cpu, decoder);
            break;
        case IType::SLTIU:
            sltiu(cpu, decoder);
            break;
        case IType::XORI:
            xori(cpu, decoder);
            break;
        case IType::SRI:
            switch (decoder.funct7() >> 1)
            {
            case IType::SRLI:
                srli(cpu, decoder);
                break;
            case IType::SRAI:
                srai(cpu, decoder);
                break;
            default:
                cpu.set_exception(exception::Exception::IllegalInstruction, decoder.insn);
                break;
            }
            break;
        case IType::ORI:
            ori(cpu, decoder);
            break;
        case IType::ANDI:
            andi(cpu, decoder);
            break;
        default:
            cpu.set_exception(exception::Exception::IllegalInstruction, decoder.insn);
            break;
        }
    }
}
```

The function above will chose an appropriate function based on `funct3`, a reminder of the instruction format:

![RVI](/posts/images/RVI-Not-CCBYNC.png)

Lets implement the `addi` instruction. The details of it can be found inside RISC-V specification inside page 18, `Integer Register-Immediate Instructions`: 
![I-IMM](/posts/images/I-IMM-Not-CCBYNC.png)

In this image we can see `I-immediate` being used. Immediate values are values that are hard coded into the instruction itself. For example, if we do:

```asm
addi a0, a0, 42
```

The immediate value is `42`. In the case of `I-immediate` instructions, the immediate is 12 bits long. It can also be negative, so we need to perform something called sign extensions. Sign extension means that if the MSB of a value is set, we need to set all the bits to the left of it to 1. For example, if we have the value `0b1000`, and we want to extend it to 32 bits, we need to set all the bits to the left of the MSB to 1, so we get `0b11111111111111111111111111111000`. The CPU you are using will most likely have sign extensions functionality we can use. Lets implement the following macros inside `Decoder.hpp`:

```cpp
#define SIGNEXTEND_CAST2(val, upcast_from) (static_cast<int64_t>(static_cast<upcast_from>(val)))

#define SIGNEXTEND_CAST(val, upcast_from)                                                          \
    (static_cast<uint64_t>(SIGNEXTEND_CAST2(val, upcast_from)))
```

We will use `SIGNEXTEND_CAST` when the result we need needs to be unsigned, and `SIGNEXTEND_CAST2` when the result needs to be signed. Now, lets add `get_imm_i()` inside `Decoder.cpp`:

```cpp
int64_t Decoder::imm_i() const
{
    return SIGNEXTEND_CAST2(insn & 0xfff00000U, int32_t) >> 20U;
}
```

Finally, lets implement `addi`:

```cpp
void addi(Cpu& cpu, Decoder& decoder)
{
    uint32_t rd = decoder.rd();
    uint32_t rs1 = decoder.rs1();
    int64_t imm = decoder.imm_i();

    cpu.regs[rd] = cpu.regs[rs1] + imm;
}
```

From now on, the basics of implementing instructions should be clear. This series will no longer go trough every line of code, and instead will focus on higher level look at the implementation details. For implementation for the rest of RVI and RVM, you can consult the RISC-V specification and my own implementation that can be found at [this folder](https://github.com/bane9/rv64gc-emu/tree/main/source/instructions).

From now on, it will be assumed RVI and RVM are fully implemented and in the next post we will set up the compiler toolchain and start writing some assembly code, after which we will run it on our emulator.