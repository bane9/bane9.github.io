---
layout: post
title:  "RISC-V Emulator part 2: Instruction fetching and decoding"
date:   2023-07-07 06:00:00 +0200
tags: ["rv64gc_emu", "RISC-V", "RISC-V Emulator"]
gh_link: rv64gc-emu
previous_post: /posts/rv64emu-part-1
next_post: /posts/rv64emu-part-3
---

As a prerequise, download RISC-V Unprivledged Specification from [here](https://riscv.org/technical/specifications/). At the time of this writing, `Volume 1, Unprivileged Specification version 20191213` will be used.

# Instruction fetching

The first step of CPU needs to do to execute an instruction is to fetch it from memory. RISC-V specification standardized instructions are almost exclusively 32-bit, with the exception of RVC (compressed instructions) that are 16-bit. For now, we will only focus on RVI (base integer instructions).

In the last post, we developed a Bus and Memory system, where the Memory system holds both the program memory and program code itself. Since we already loaded the program into RAM, we can start fetching instructions from it.

So lets create a `Cpu` class that will encapsulate the CPU state and logic. We will also create a `Cpu::fetch` method that will fetch the next instruction from memory.

Inside `source/include/cpu.hpp`:

```cpp
#include "bus.hpp"
#include <cstdint>

class Cpu
{
public:
    Cpu(Bus *bus, uint64_t pc = 0x80000000U);
    ~Cpu() = default;

    uint32_t fetch();
    bool do_insn();
private:
    Bus *bus;
    uint64_t registers[32]; // RISC-V Specifies 32 integer registers
    uint64_t pc;
};
```

Inside `source/cpu.cpp`:

```cpp
#include "cpu.hpp"

Cpu::Cpu(Bus *bus, uint64_t pc) : bus(bus), pc(pc)
{
}

uint32_t Cpu::fetch()
{
    return bus->read(pc, 32);
}

bool do_insn()
{
    uint32_t insn = fetch();

    return true;
}
```

Now, let's add the CPU to our `main.cpp`:

```cpp
#include <iostream>
#include "helper.hpp"

int main(int argc, char** argv)
{
    if (argc < 2)
    {
    std::cout << "Usage: " << argv[0] << " <path to binary>" << std::endl;
    return 1;
    }

    std::vector<uint8_t> binary = helper::load_file(argv[1]);

    Bus bus;

    std::unique_ptr<RAM> ram = std::make_unique<RAM>(0x80000000, 0x81000000); // 16MiB
    ram->set_data(binary);

    bus.add_peripheral(std::move(ram));

    Cpu cpu(&bus);

    bool run = true;

    while (run)
    {
        run = cpu.do_insn();
    }
}
```

# Instruction Decoding

Now that we have the instruction fetched, we need to decode it. The instruction format for RVI can be seen in the RISC-V specification at `2.2 Base Instruction Formats`. The instruction format is as follows:

![RVI](/posts/images/RVI-Not-CCBYNC.png)

In the image we can see that the instruction consists of several different fields. Breaking down, we can see that the instruction consists of:

- opcode: This is the opcode of the instruction. It is used to determine what instruction is being executed.
- rd: This is the destination register. It is used to determine where the result of the instruction is going to be stored.
- funct3: This is the function field. It is used to determine what variant of the instruction is being executed.
- funct7: This is the second function field. It is used to determine what subvariant of funct3 of the instruction is being executed.
- rs1: This is the first source register. It is used to determine the first operand of the instruction.
- rs2: This is the second source register. It is used to determine the second operand of the instruction.

Also we can see instruction can be different types. Each type has it's fields defined differently as can be seen in the picture, which we need to take into account when decoding an instruction. We can determine what type an instruction is by looking at the opcode.

To decode the instruction, lets create a decoder class.

Inside `source/include/decoder.hpp`:

```cpp
#include <cstdint>

class Decoder
{
public:
    Decoder(uint32_t insn);
    ~Decoder() = default;

    uint32_t get_opcode() const;
    uint32_t get_rd() const;
    uint32_t get_funct3() const;
    uint32_t get_funct7() const;
    uint32_t get_rs1() const;
    uint32_t get_rs2() const;
    uint32_t get_insn() const;
    uint32_t get_len() const;
private:
    uint32_t insn;
};
```

Inside `source/decoder.cpp`:

```cpp
#include "decoder.hpp"

Decoder::Decoder(uint32_t insn) : insn(insn)
{
}

uint32_t Decoder::get_opcode() const
{
    return insn & 0x7FU;
}

uint32_t Decoder::get_rd() const
{
    return (insn >> 7) & 0x1FU;
}

uint32_t Decoder::get_funct3() const
{
    return (insn >> 12) & 0x7U;
}

uint32_t Decoder::get_funct7() const
{
    return (insn >> 25) & 0x7FU;
}

uint32_t Decoder::get_rs1() const
{
    return (insn >> 15) & 0x1FU;
}

uint32_t Decoder::get_rs2() const
{
    return (insn >> 20) & 0x1FU;
}

uint32_t Decoder::get_insn() const
{
    return insn;
}

// This will be explained in detail at a later post:
uint32_t Decoder::get_len() const
{
    switch (insn & 0x3U)
    {
    case 0x00U:
    case 0x01U:
    case 0x02U:
        return 2;
    default:
        return 4;
    }
}
```

Now, back inside `Decoder.hpp`, lets define enums for different instruction types and functions:

```cpp
enum class OpcodeType : uint64_t
{
    LOAD = 0x03,

    AUIPC = 0x17,
    LUI = 0x37,

    JALR = 0x67,
    JAL = 0x6f,

    I = 0x13,
    S = 0x23,
    R = 0x33,
    B = 0x63,

    I64 = 0x1b,
    R64 = 0x3b,
};

struct LOADType
{
    enum funct3 : uint64_t
    {
        LB = 0x00,
        LH = 0x01,
        LW = 0x02,
        LD = 0x03,
        LBU = 0x04,
        LHU = 0x05,
        LWU = 0x06
    };
};

struct IType
{
    enum funct3 : uint64_t
    {
        ADDI = 0x00,
        SLLI = 0x01,
        SLTI = 0x02,
        SLTIU = 0x03,
        XORI = 0x04,
        SRI = 0x05,
        ORI = 0x06,
        ANDI = 0x07
    };

    enum SRIfunct7 : uint64_t
    {
        SRLI = 0x00,
        SRAI = 0x10
    };
};

struct SType
{
    enum funct3 : uint64_t
    {
        SB = 0x00,
        SH = 0x01,
        SW = 0x02,
        SD = 0x03
    };
};

struct RType
{
    enum funct3 : uint64_t
    {
        ADDMULSUB = 0x00,
        SLLMULH = 0x01,
        SLTMULHSU = 0x02,
        SLTUMULHU = 0x03,
        XORDIV = 0x04,
        SR = 0x05,
        ORREM = 0x06,
        ANDREMU = 0x07
    };

    enum ADDSUBfunct7 : uint64_t
    {
        ADD = 0x00,
        MUL = 0x01,
        SUB = 0x20
    };

    enum SLLMULHfunct7 : uint64_t
    {
        SLL = 0x00,
        MULH = 0x01,
    };

    enum SLTMULHSUfunct7 : uint64_t
    {
        SLT = 0x00,
        MULHSU = 0x01,
    };

    enum SLTUMULHUfunct7 : uint64_t
    {
        SLTU = 0x00,
        MULHU = 0x01,
    };

    enum XORDIVfunct7 : uint64_t
    {
        XOR = 0x00,
        DIV = 0x01,
    };

    enum SRfunct7 : uint64_t
    {
        SRL = 0x00,
        DIVU = 0x01,
        SRA = 0x20
    };

    enum ORREMfunct7 : uint64_t
    {
        OR = 0x00,
        REM = 0x01,
    };
    
    enum ANDREMUfunct7 : uint64_t
    {
        AND = 0x00,
        REMU = 0x01,
    };
};

struct BType
{
    enum funct3 : uint64_t
    {
        BEQ = 0x00,
        BNE = 0x01,
        BLT = 0x04,
        BGE = 0x05,
        BLTU = 0x06,
        BGEU = 0x07
    };
};

struct I64Type
{
    enum funct3 : uint64_t
    {
        ADDIW = 0x00,
        SLLIW = 0x01,
        SRIW = 0x05
    };

    enum SRIWfunt7 : uint64_t
    {
        SRLIW = 0x00,
        SRAIW = 0x20
    };
};

struct R64Type
{
    enum funct3 : uint64_t
    {
        ADDSUBW = 0x00,
        SLLW = 0x01,
        DIVW = 0x04,
        SRW = 0x05,
        REMW = 0x06,
        REMUW = 0x07
    };

    enum ADDSUBWfunct7 : uint64_t
    {
        ADDW = 0x00,
        MULW = 0x01,
        SUBW = 0x20
    };

    enum SRWfunct7 : uint64_t
    {
        SRLW = 0x00,
        DIVUW = 0x01,
        SRAW = 0x20
    };
};
```

More information about the instruction types and functions can be found in the RISC-V specification at page 130, at tables `RV32I Base Instruction Set` and `RV64I Base Instruction Set (in addition to RV32I)`.

Then, back in `Cpu::do_insn`, we can decode the instruction:

```cpp
bool do_insn()
{
    uint32_t insn = fetch();
    Decoder decoder = Decoder(insn);

    return true;
}
```

In the next post, we will implement the instruction execution.