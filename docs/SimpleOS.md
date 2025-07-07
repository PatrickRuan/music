# Simple OS on macOS

This document outlines a minimal approach to build and run a simple toy operating system using tools that work on macOS.

## 1. Planning and Goals

1. Start with a basic "hello world" kernel that boots and prints text.
2. Run the kernel inside a virtual machine such as QEMU. This avoids needing extra hardware.
3. Keep the code minimal and readable. We use the `nasm` assembler and `ld` linker included with the toolchain.

## 2. Environment Setup

1. **Homebrew** – install the necessary packages:
   ```bash
   brew install qemu nasm
   ```
2. Ensure you have a working C compiler (Xcode or Command Line Tools provides `clang`).

## 3. Project Structure

```
myos/
├── boot.s      # assembly bootstrap
├── kernel.c    # minimal C kernel
├── linker.ld   # linker script
└── Makefile    # build rules
```

## 4. Bootstrapping Code (`boot.s`)

This code sets up a 32‑bit environment and jumps to `kernel_main` defined in C:

```asm
[bits 32]
extern kernel_main
multiboot_header:
    dd 0x1BADB002            ; magic
    dd 0x00                  ; flags
    dd -(0x1BADB002)         ; checksum
[section .text]
    global _start
_start:
    cli
    mov esp, stack_space + stack_size
    push ebx                 ; multiboot info
    push eax                 ; magic value
    call kernel_main
.hang:
    hlt
    jmp .hang
section .bss
stack_space:
    resb stack_size
stack_size equ 4096
```

## 5. Kernel (`kernel.c`)

A minimal kernel prints a string using VGA text memory:

```c
#include <stdint.h>

static volatile char* video = (char*)0xB8000;

void kernel_main(void) {
    const char* msg = "Hello, OS!";
    for (int i = 0; msg[i]; ++i) {
        video[i * 2] = msg[i];
        video[i * 2 + 1] = 0x07;
    }
    while (1) { __asm__("hlt"); }
}
```

## 6. Linker Script (`linker.ld`)

```
ENTRY(_start)
SECTIONS {
  . = 1M;             /* load at 1 MB */
  .text : { *(.text*) }
  .bss : { *(.bss*) }
}
```

## 7. Build System (`Makefile`)

```
CC=clang
AS=nasm
LD=ld
CFLAGS=-m32 -ffreestanding -O2

all: myos.bin

myos.bin: boot.o kernel.o linker.ld
$(LD) -m elf_i386 -T linker.ld -o myos.bin boot.o kernel.o

boot.o: boot.s
$(AS) -f elf32 boot.s -o boot.o

kernel.o: kernel.c
$(CC) $(CFLAGS) -c kernel.c -o kernel.o

run: all
qemu-system-i386 -kernel myos.bin

clean:
rm -f *.o myos.bin
```

## 8. Building and Running

1. Open Terminal in the project folder.
2. Run `make run`. QEMU starts and you should see "Hello, OS!" in a window.

## 9. Next Steps

From here, you can explore keyboard input, memory management, and multitasking. Many hobby OS resources and tutorials are available for deeper topics.

