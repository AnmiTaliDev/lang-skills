# Cinder CLI Reference

The compiler is invoked as `cinder` (a wrapper around `main.rb` /
`ruby main.rb` if running from a source checkout). Full pipeline:
lexer -> parser -> loader -> sema -> (codegen if building).

## Commands

| Command | Action |
|---------|--------|
| `check <file>` | lex, parse, load modules, run static analysis only |
| `build <file>` | `check`, then generate output (LLVM IR / asm / obj / bin / kernel) |
| `tokens <file>` | dump the token stream (debug) |
| `ast <file>` | dump the AST (debug) |

## Options

| Option | Meaning |
|--------|---------|
| `-I <dir>` | add a module search directory (repeatable) |
| `--target=<arch>` | target architecture; default `x86_64`. Known: `x86_64`, `x86_64-freestanding` |
| `--emit=llvm\|asm\|obj\|bin\|kernel` | output format; default `llvm` |
| `--mode=debug\|release` | build mode; default `debug` |
| `--linker-script=<path>` | linker script, used with `--emit=kernel` |
| `--boot=<file.s>` | raw assembly boot stub linked into the kernel (e.g. multiboot entry) |
| `--entry=<name>` | kernel entry symbol; default `_start` |
| `-o <file>` | output file (default derived from input: `.ll`/`.s`/`.o`/bare name) |
| `-v`, `--verbose` | keep intermediate `.ll`/`.s`/`.o` files, print toolchain commands |
| `-h`, `--help` | show help |
| `--version` | print compiler version |

## Examples

```bash
# static-check only, no codegen
cinder check main.cnd

# emit LLVM IR text
cinder build main.cnd --emit=llvm -o main.ll

# emit a native executable (links libc)
cinder build main.cnd --emit=bin -o main

# add a module search dir (for `use "queue.cnd";` etc.)
cinder build main.cnd --emit=bin -o main -I lib

# release mode
cinder build main.cnd --emit=bin --mode=release -o main

# freestanding kernel image (bare metal, no libc)
cinder build main.cnd --target=x86_64-freestanding \
    --emit=kernel --linker-script=linker.ld --boot=boot.s -o kernel.bin
```

## Bare-metal (kernel) builds

`--emit=kernel` produces a freestanding ELF with no libc. Needs:
- `--boot=<file.s>` — a raw assembly entry stub (e.g. multiboot header +
  stack setup + `call kernel_main`)
- `--linker-script=<path>` — places sections/base address
- An `export fn kernel_main() { ... }` in your `.cnd` file — it must be
  `export`ed because the boot stub calls it by symbol name
- Only `std/x86.cnd` (port I/O: `outb`/`inb`/etc.) and `asm("...")` inside
  `unsafe` are available; no `io`/`alloc`/`vec`/`string` (they need libc)

Minimal example set (from `examples/kernel/` in the repo):

`boot.s`:
```gas
.section .multiboot
.align 4
.long 0x1badb002
.long 0
.long -(0x1badb002)

.section .text
.global _start
_start:
    movl $stack_top, %esp
    call kernel_main
hang:
    hlt
    jmp hang

.section .bss
.align 16
stack_bottom:
    .skip 4096
stack_top:
```

`linker.ld`:
```ld
ENTRY(_start)
SECTIONS {
    . = 1M;
    .text : { *(.multiboot) *(.text) }
    .bss  : { *(.bss) }
}
```

`main.cnd`:
```rust
use "std/x86.cnd";

unsafe fn uart_write_byte(b: u8) {
    while (inb(0x3FD) & 0x20) == 0 { asm("nop"); }
    outb(0x3F8, b);
}

export fn kernel_main() {
    unsafe {
        uart_write_byte('H' as u8);
        loop { asm("hlt"); }
    }
}
```

Build & run under QEMU:
```bash
ruby main.rb build main.cnd --emit=kernel \
    --boot=boot.s --linker-script=linker.ld -o kernel
qemu-system-i386 -nographic -kernel kernel
```

`#[target("x86_64-freestanding")]` restricts a declaration to only compile
for the freestanding target, letting one file support both hosted and
bare-metal builds.
