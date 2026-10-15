---
title: How I ran a 23KiB program on a 16KiB Flash device
date: 2026-10-15 21:30:00 +0200
categories: [Embedded, Compression]
tags: [riscv, embedded, compression, sseocb]
---

## Preface

While modern computers usually have gigabytes of memory, small microcontrollers
operate with megabytes or even kilobytes. For example, the Adafruit QT Py CH32V203 has
only 32 KiB of nominal flash memory, which dictates the size of the programs that can be
run on it. If a larger program is needed, you only have one option: upgrade your
device.

While this is fine for most people, companies that ship millions of such devices (for
example, in embedded electronics) have to spend millions of dollars to upgrade every
single device.

## Hardware solution
To solve this issue, data compression can be used—specifically, compression of the
program code. Some devices have hardware support for this. For example, IBM's
`CodePack` was developed in the 1990s for this exact purpose, and it solves the size
issue quite well: programs compressed with it show up to a 40% reduction in size while
maintaining roughly the same level of performance.

However, an important detail is that this is a *hardware* solution, which means that
any device not supporting hardware (de)compression will not be able to use it. There
are many different kinds of devices, and virtually no low-cost commodity microcontrollers
support hardware decompression today.

## Software-side executor of compressed binaries
[SSEOCB](https://github.com/lis05/sseocb) is a new open-source system which, as suggested by the name, is a software solution that
supports the execution of compressed programs. The core idea is to compress the program
(known as guest code) and then have a dedicated interpreter execute it instruction by
instruction, decompressing required parts of the program at each step using a
decompressor also included in the final binary.

![Core idea](/assets/sseocb-core-idea.drawio.svg)

- Let `O` be the size of the original, uncompressed program.
- Let `I` be the size of the interpreter.
- Let `D` be the size of the decompressor.
- Let `C` be the size of the compressed program.

Then, `I + D + C < O` must be true for this idea to actually save memory.

### How it works
Several architectural mechanisms make SSEOCB possible.

First of all, RISC-V was chosen as the target architecture. It has a very simple
instruction set (40 instructions if using no extensions), which makes the interpreter
very compact.

RISC-V is also well-suited for this because it features two base variations:
- RV32E (with 16 registers)
- RV32I (with an additional 16 registers, totaling 32).

RV32I differs from RV32E only in how many registers can be addressed in an instruction,
while the instruction encoding remains otherwise identical. Therefore, in SSEOCB, guest code
and the interpreter use separate sets of 16 registers. Both are compiled for RV32E, but
the interpreter code is then processed by a tool that remaps its used registers into the
higher half of the register file (x16–x31).

The most important aspect of SSEOCB is how the compressed program is addressed. The `.text` section is
compressed and linked at address `0xfff00000` (1 MiB away from the maximum 4 GiB
address). No other code sits above that address, so when the interpreter sees a memory
access or a jump to an address starting with `0xfff`, it knows the target is inside the compressed
program and invokes the decompressor. The compressed program isn't actually placed
at `0xfff00000`—it is later placed in physical memory, but the rest of the code sees
it as if it were at the end of the address space.

SSEOCB also handles jumps between guest code (which is compressed) and native code
(which is left uncompressed for performance reasons, or is external code). Such jumps
require the interpreter to briefly yield control of the program and cost
about 60 bytes of stack space per `native -> guest` jump, but this is a rare scenario
that only happens when a function is passed to an external library or peripheral as a callback.

The whole pipeline involves extensive linking and patching of object files.
![Pipeline](/assets/sseocb-pipeline.drawio.svg)

## Demo
As a proof of concept, I was able to run uLisp Zero, a compact bare-metal interpreter for the Lisp language, on
a simulated device with just 16 KiB of memory. The original binary would not even
link, as it was too large (~23 KiB of Flash space) for the linker to fit it. With
SSEOCB (and its `blaris` compressor), I was able to fit it in a little less than 16
KiB (15.8 KiB) and actually run it!
![Demo](/assets/demos-ulisp.png)

The interpreter with decompressor weighs about 5 KiB, and the compressed program
(or rather its `.text` section) is at 9.5 KiB (which is a 55.6% reduction from
the original 21.6 KiB `.text` section!).

## Limitations
My ultimate goal for SSEOCB is to seamlessly run any program without
having to tinker with its source code. Right now this is mostly satisfied; however,
there are some limits—for example, static symbols cannot be passed from guest code to
native code as they are treated differently by the compiler and are not addressed as
symbols, but rather as offsets within the `.text` section.

SSEOCB can introduce a substantial performance impact, reaching slowdown factors as
large as `70,000x`–`100,000x` in extreme cases. It is important to note that this slowdown
comes from the fact that SSEOCB has to simulate instructions, as well as from
using `blaris`, a generic decompressor that was not optimized for this project or for
instruction streams. Much faster options can be used: for example, using the `graemi` compressor
(with 1% checkpoints) runs with only a `~1,100x` slowdown (executing the full uLisp demo in about 2 seconds),
and uncompressed virtualization incurs only an `~80x` overhead.

Furthermore, performance-critical hot paths can be extracted into native code, while leaving the other
90% of the code (which is almost always not performance-critical) compressed. Still, this is an important
trade-off, and optimizing decompressor hot loops and developing specialized domain-specific compression strategies
remain an ongoing focus.

The full codebase and reproduction steps are open source on GitHub: [github.com/lis05/sseocb](https://github.com/lis05/sseocb).

> P.S.
> This is the first post in the series about SSEOCB. More are to come!
