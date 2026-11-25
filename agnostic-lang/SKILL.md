---
name: agnostic-lang
description: >
  Write, edit, review, or explain code in the Agnostic programming language
  (.agn files, or the "agnostic" compiler). Use when the user asks for Agnostic
  code, mentions .agn files, asks to port or compare code to Agnostic, or asks
  about its syntax, comptime, standard library (stdio, math, string, os,
  Result, Option), memory modes, backends, or CLI flags. Do not confuse it with
  Go, even though the syntax looks similar.
---

# Agnostic

Agnostic is a compiled systems language with LLVM and libgccjit backends and
no libc dependency in the produced binaries. The syntax looks like Go, but the
language is much smaller and the parser has hard limits. Do not assume that Go
syntax or Go library names work.

This skill is based on the source at
https://github.com/agnosticlang/agnostic, commit `2617cc7` (2026-10-07). It
was derived by reading the parser, type checker, code generator, docs, stdlib,
and examples. Where the upstream docs and the source differ, the skill follows
the source.

## Minimal program

```agn
package main

import "stdio"

func main() {
    stdio.PrintlnStr("Hello, World!")
}
```

```sh
agnostic hello.agn --output=hello
./hello
```

## Rules that are easy to get wrong

1. **Newlines end statements.** Do not write `;`. Keep every expression and
   every call argument list on one line.
2. **Not Go:** no `:=`, `+=`, `++` as increment, `else if`, `switch`, `range`,
   three-part `for`, slices, maps, interfaces, multiple return values, `nil`,
   `const`, `defer`, goroutines, or block comments. `++` concatenates strings.
3. **Top level has only `package`, then `import`, then `struct` and `func`.**
   There are no global variables.
4. **`main` is `func main()`** with no parameters and no return type. Use
   `os.Exit(code)` for an exit code and `os.ArgCount` / `os.Arg` for arguments.
5. **Conditions must be `bool`.** Write `if n != 0`, not `if n`.
6. **`&&` and `||` do not short-circuit** in the default `llvm` backend. Both
   sides are always evaluated. Never write `i < n && arr[i] > 0` as a bounds
   guard. Nest `if` blocks.
7. **There are no bounds checks** on arrays or string literals.
8. **`==` on strings compares addresses.** Use `string.compare(a, b) == 0`.
9. **Only one level of access.** `a.b`, `a.b(...)`, `a[i]`, and `f(...)` work
   when `a` is a plain name. `a.b.c`, `arr[i].x`, `f().x`, and `a.m().n()` do
   not. Copy into a local variable first.
10. **Do not index a string variable.** `s[i]` is rejected. Use
    `string.charAt(s, i)`, which returns `Option<int>`. Only a string literal
    can be indexed.
11. **Arrays exist only as local variables:** `var a [8]int`. No array
    literals, array parameters, array fields, or array return types. Pass
    `&a` where a `*T` is expected.
12. **Imported modules export functions only.** A `struct` declared in an
    imported file is not loaded. Declare structs in the main source file.
13. **Parenthesize bitwise and shift expressions** when they are combined with
    comparisons or `&&`. See `references/syntax.md#operators`.

## Reference files

- `references/syntax.md`: types, conversions, operators, control flow,
  pointers, arrays, structs, closures, comptime, modules. Read it before
  writing anything beyond a trivial program.
- `references/stdlib.md`: every function in `stdio`, `math`, `string`, `os`,
  plus `Result` and `Option`. Check it before using a library function.
- `references/cli.md`: compiler flags, backends, memory modes, targets, and
  building the compiler.

## Workflow

1. Check each library call against `references/stdlib.md`.
2. Write the code, then check it against the rules above.
3. If the compiler is available, compile and run the program. Type checking
   happens as part of compilation. There is no separate check command.
