---
layout: post
title:  "RISC-V Emulator part 1: Getting started"
date:   2023-06-25 16:00:00 +0200
tags: ["rv64gc_emu", "RISC-V", "RISC-V Emulator"]
gh_link: rv64gc-emu
previous_post: /posts/riscv-emulator-from-scratch
next_post: /posts/rv64emu-part-2
---

In this post, we will set up the project environment and write the basic code needed to get started. This project will be written in C++, and will be using CMake as the build system. An Ubuntu machine will be assumed for some of the steps, but the steps should be similar for other Linux distributions and macOS.

# Setting up the project

First, get the basic dependencies:

```bash
sudo apt install build-essential cmake git
```

Then, create a new directory for the project and initialize a git repository:

```bash
mkdir emulator
cd emulator
git init
```

Now, create the following folder structure:

```
emulator/
├─ source/
│  ├─ include/
│  ├─ instructions/
│  │  ├─ include/
│  ├─ peripherals/
│  │  ├─ include/
├─ main.cpp
├─ CMakeLists.txt
```

Then add the following to CMakeLists.txt:

```cmake
cmake_minimum_required(VERSION 3.16)
project(emulator)

set(INCLUDE_DIRS
  ${PROJECT_SOURCE_DIR}/source/include/
  ${PROJECT_SOURCE_DIR}/source/instructions/include/
  ${PROJECT_SOURCE_DIR}/source/peripherals/include/
)

set(SRC_FILES
  ${PROJECT_SOURCE_DIR}/main.cpp
)

add_executable(${PROJECT_NAME} ${SRC_FILES})

target_include_directories(${PROJECT_NAME} PUBLIC ${INCLUDE_DIRS})
```

# Starting with basics
## Pre CPU initialization

Just like any program we run, it needs to be loaded from somewhere. In our case, we will be loading compiled RISC-V binaries and putting them into emulators RAM peripheral. To start this off, lets first create a set of header only helper functions that will do some common functionality. To start, make a `helper.hpp` inside `source/include/` and add the following:

```cpp
#pragma once

#include <fstream>
#include <vector>

namespace helper
{
inline std::vector<uint8_t> load_file(const char* file_path)
{
  std::ifstream input(filename, std::ios::binary);
  return std::vector<uint8_t>{std::istreambuf_iterator<char>(input),
                              std::istreambuf_iterator<char>()};
}

}
```

Now, in `main.cpp`, lets load that file into a vector by looking at `argv[1]`:

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
}
```

## Bus and peripherals
### Peripherals

In the previous post, this image was displayed:

![RV64GC Emu System Architecture](/posts/images/system_arch.png#center)

Right now, we wan't to implement the `System Bus`. The bus itself acts as a communication line between the CPU and the peripherals, and between the peripherals and other peripherals. The way it works is that each peripheral will have a start and an end address. For example, in the following image:

![RV64GC Emu System Architecture](/posts/images/mmio.png#center)

Here we can see that each peripheral has it's start and end address. When we, for example, want to talk to Peripheral 2, we need to specify an address that is between `0x5000` and `0x9000`. As such, if we want to talk to any other peripheral we need to specify it's own address range. This is called "Memory Mapped IO" or MMIO for short.

Another aspect of communication within the bus is that each read/write operation also needs to specify the amount of bits it will transfer. Operation sizes can be 8, 16, 32 and 64-bit, given we are designing a 64-bit CPU.

Having that in mind, let's design our "Peripheral" object. In this series we will use inheritence to create different types of peripherals that we will later add on the bus.

In `source/peripherals/include/peripheral.hpp`:

```cpp
#pragma once

#include <cstdint>

class Peripheral
{
  public:
  Peripheral(uint64_t start_addr, uint64_t end_addr)
    : start_addr_(start_addr), end_addr_(end_addr)
  {
  }

  virtual ~Peripheral() = default;

  virtual uint64_t read(uint64_t addr, uint8_t size) = 0;
  virtual void write(uint64_t addr, uint64_t data, uint8_t size) = 0;

  void get_start_addr() const { return start_addr_; }
  void get_end_addr() const { return end_addr_; }

  protected:
  uint64_t start_addr_;
  uint64_t end_addr_;
};
```

### Bus
Now, let's create the bus itself. In `source/peripherals/include/bus.hpp`:

```cpp
#pragma once

#include <cstdint>
#include <memory>
#include <vector>
#include "peripheral.hpp"

class Bus
{
  public:
  Bus() = default;
  ~Bus() = default;

  void add_peripheral(std::unique_ptr<Peripheral> peripheral);

  Peripheral* get_peripheral(uint64_t addr);

  void write(uint64_t addr, uint64_t data, uint8_t size);
  uint64_t read(uint64_t addr, uint8_t size);

  private:
  std::vector<std::unique_ptr<Peripheral>> peripherals_;
};
```

And in `source/peripherals/bus.cpp`:

```cpp
#include "bus.hpp"

void Bus::add_peripheral(std::unique_ptr<Peripheral> peripheral)
{
  peripherals_.push_back(std::move(peripheral));
}

Peripheral* Bus::get_peripheral(uint64_t addr)
{
  for (auto& peripheral : peripherals_)
  {
    if (addr >= peripheral->get_start_addr() &&
        addr <= peripheral->get_end_addr())
    {
      return peripheral.get();
    }
  }

  return nullptr;
}

void Bus::write(uint64_t addr, uint64_t data, uint8_t size)
{
  Peripheral* peripheral = get_peripheral(addr);
  if (peripheral)
  {
    peripheral->write(addr, data, size);
  }
}

uint64_t Bus::read(uint64_t addr, uint8_t size)
{
  Peripheral* peripheral = get_peripheral(addr);
  if (peripheral)
  {
    return peripheral->read(addr, size);
  }

  return 0;
}
```

## RAM peripheral
Now that we have our bus and peripheral objects, let's create a RAM peripheral. This peripheral will be used to store the binary we loaded in the beginning of the program. In `source/peripherals/include/ram.hpp`:

```cpp
#pragma once

#include <cstdint>
#include <vector>
#include "peripheral.hpp"

class RAM : public Peripheral
{
  public:
  RAM(uint64_t start_addr, uint64_t end_addr);
  ~RAM() = default;

  uint64_t read(uint64_t addr, uint8_t size) override;
  void write(uint64_t addr, uint64_t data, uint8_t size) override;

  void set_data(const std::vector<uint8_t>& data);

  private:
  std::vector<uint8_t> data_;
};
```

And in `source/peripherals/ram.cpp`:

```cpp
#include "ram.hpp"

RAM::RAM(uint64_t start_addr, uint64_t end_addr)
  : Peripheral(start_addr, end_addr)
{
}

void RAM::set_data(const std::vector<uint8_t>& data)
{
  data_ = data;
  data.resize(end_addr_ - start_addr_);
}

uint64_t RAM::read(uint64_t addr, uint8_t size)
{
  addr -= start_addr_; // We need to offset the address to the actual start of the RAM data

  switch (size)
  {
    case 8:
      return data_[addr];
    case 16:
      return *reintrepret_cast<uint16_t*>(data_ + addr);
    case 32:
      return *reintrepret_cast<uint32_t*>(data_ + addr);
    case 64:
      return *reintrepret_cast<uint64_t*>(data_ + addr);
  };
}

void RAM::write(uint64_t addr, uint64_t data, uint8_t size)
{
  addr -= start_addr_;

  switch (size)
  {
    case 8:
      data_[addr] = data;
      break;
    case 16:
      *reintrepret_cast<uint16_t*>(data_ + addr) = data;
      break;
    case 32:
      *reintrepret_cast<uint32_t*>(data_ + addr) = data;
      break;
    case 64:
      *reintrepret_cast<uint64_t*>(data_ + addr) = data;
      break;
  }
}
```

Now, let's add what we have in the CMakeLists.txt within SRC_FILES:

```cmake
set(SRC_FILES
  ...
  peripherals/include/peripheral.hpp
  peripherals/include/bus.hpp
  peripherals/include/ram.hpp
  peripherals/bus.cpp
  peripherals/ram.cpp
)
```

## Tying things together

Now that we have our RAM peripheral, let's add it to our bus. In `main.cpp`:

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
}
```

Note that the term `MiB`. We need to understand the disctinction between `MiB` and `MB`. `MiB` stands for Mebibyte, and `MB` stands for Megabyte. The difference is that `MiB` is 1024 * 1024 bytes, and `MB` is 1000 * 1000 bytes. In this series we will use `MiB` to avoid confusion.

In the first several parts of this tutorial series, code will be explained more thoughroughly as to make sure we can start off with a common basic undestanding of how the whole CPU and the rest of the system work. In the later parts, the focus will shift more towards the specification, CPU and system architecture, while the code is going to be left to the viewer to implement themselves. 

In the next part, we will start implementing the CPU itself.