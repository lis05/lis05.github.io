---
title: bbb
github: https://github.com/lis05/bbb
date: 2026-04-04
description: "A low-level, memory-first 64-bit programming language targeting x86-64 Unix platforms."
---

bbb (pronounced like bee bee bee, but fast) is my latest and most complex project
related to programming languages.

> A low-level, memory-first 64-bit programming language targeting x86-64 unix
platforms compliant with the System V ABI.

It is a compiled programming language that is built on top of the concepts that are
heavily used in assembly programming - i.e. the idea that everything is just plain
memory, but what you do with that memory defines what it is.

bbb is designed to sit between C and assembly - allowing for high-level cosntructs, while at the same time providing the programmer with access to low-level functionality. 

A (somewhat explicit) sample of bbb code that prints Fibonacci numbers:
```
extern printf

fib: fn(n: m4 align4 #int) -> m8 #int {
    if n m4==m4 0?m4 {
        ret 0
    }
    if n m4==m4 1?m4 {
        ret 1
    }
    ret call m8 #int fib(n m4-m4 1?m4) + call m8 #int fib(n m4-m4 2?m4);
}

main: global fn() -> m4 {
    i: alias rax
    i m4=m4 0?m4;

    loop {
        if i m4==m4 10?m4 {
            break;
        }

        call printf("%zu\n", call fib(i: m4 align4): m8);

        i m4=m4 m4++i;
    }

    ret 0;
}
```

---

Currently, bbb is not yet fully (nor is close to being fully) functional. It can compile just very basic parts of the language, but I am constantly working on it. One day, the world will welcome yet another programming language!

What has already been implemented? Parser is 100% complete, but in regards to codegen:
- ✅ - global variable declarations
- ✅ - layout declarations
- ✅ - extern declarations
- ⚠️ - function declarations (return type is not yet processed)
- ✅ - local variable declarations
- ✅ - register aliases
- ❌ - if statement
- ❌ - labels
- ❌ - goto statement
- ❌ - break / continue
- ❌ - return statement
- ❌ - avoid blocks
- ❌ - expressions
  - ⚠️ - assignment (extremely simple case implemented)
  - ✅ - literals
  - ❌ everything else
- ✅ - NASM blocks
- ⚠️ - variable palceholders in NASM blocks (not all cases)