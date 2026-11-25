---
name: cinder-lang
description: Write, edit, review, or explain code in the Cinder programming language (files with the .cnd extension, or the "cinder" compiler/CLI). Use this whenever the user asks for Cinder code, mentions .cnd files, asks to port/compare code to Cinder, or asks about Cinder syntax, its standard library (std/io.cnd, std/vec.cnd, std/string.cnd, std/alloc.cnd, std/file.cnd, std/core/*), or the `cinder check`/`cinder build` CLI. Cinder is a small C-like systems language that compiles to LLVM IR. Do not confuse it with C, Rust, Zig, or any other language just because the syntax looks similar; always follow the exact rules in this skill instead of guessing from those languages.
---

# Cinder

Cinder is a minimalist systems programming language (LLVM-IR backend, C-ABI
compatible, bare-metal capable). It looks like a cousin of Rust/Zig/C but has
its own, much smaller rule set.
Do not import syntax or semantics from Rust/Zig/Go/C on faith. When unsure,
check `references/spec.md` rather than guessing.

This skill is based on the compiler and stdlib source at
https://github.com/z3nnix/cinder, commit `a94bfdf` (2026-09-16, version file
`v0.2.1-bootstrap (b20260814)`). If something here conflicts with general
priors about C-like languages, prefer what is written here.

## Quick orientation

- File extension: `.cnd`
- Entry point: `fn main() -> i32 { ... }` (or a void `fn main() { }`, or the
  C-style `fn main(argc: i32, argv: **u8) -> i32 { }` for CLI args)
- Every statement ends with `;`. Struct fields / enum variants end with `;`
  or `,` (trailing separator allowed).
- Naming: `snake_case` for variables/functions, `PascalCase` for
  structs/enums.
- No classes, no methods, no generics, no macros, no templates, no
  operator overloading, no exceptions. "Methods" are just functions that
  take a pointer to the struct as the first argument (see
  `references/syntax.md#structs`).
- Use `?T` (optional) and `!T` (error union) for missing values and failures.
  Raw pointers (`*T`) can be `null` and are used without any special block.
- Import stdlib/local modules with `use "path.cnd";` at the top of the file.
  `std/` is always on the search path (e.g. `use "std/io.cnd";`).

## Minimal working program

```rust
use "std/io.cnd";

fn main() -> i32 {
    println("Hello, world!");
    return 0;
}
```

Build/check it with the CLI (see `references/cli.md` for all flags):

```bash
cinder check hello.cnd                       # lex+parse+sema only
cinder build hello.cnd --emit=bin -o hello    # produce a native binary
./hello
```

## Core rules to never violate

1. **Immutability by default.** `let x = 1;` is immutable. Use
   `let mut x = 1;` if it will be reassigned or have a field mutated.
2. **No `NULL` as a failure value.** A function that might not return a value
   returns `?T`; a function that might fail returns `!T` (usually with an
   enum error variant). Unwrap with `let v = expr else { ... /* must
   return/break/panic */ };` or propagate with `expr?` (only valid inside a
   function whose return type is itself `!T`).
3. **Do not write `unsafe`.** `unsafe { }` blocks and `unsafe fn` are
   deprecated, have no effect, and produce a compiler warning. Pointer
   dereference, arithmetic, indexing, slicing, casts, `asm(...)`, and any
   function call are allowed everywhere. The compiler does not protect pointer
   code, so check `null` and lengths in the code itself.
4. **Struct field mutation needs `mut` on the binding**, and you assign the
   *field*, not the whole struct: `let mut p = Point{...}; p.x = 5;` — you
   cannot do `p == q` (no aggregate struct comparison).
5. **Strings are `[]u8` (byte slices), not a distinct String type.** The
   growable, heap-owned type is `std/string.cnd`'s `String`, built on
   `std/vec.cnd`'s `Vec`. Indexing a string/slice returns a `u8` byte.
6. **`switch` has no fallthrough.** Cases are alternatives, matched by
   integer, integer range (`a..b`), enum variant (`Enum.Variant` or the
   `.Variant` shorthand when the type is known), or string. `else` is the
   catch-all.
7. **Casts use `as`.** This covers numeric and pointer casts.
8. **Constant out-of-bounds indexes are not compile errors.** `a[9]` on a
   3-element array passes `check`. Debug builds check indexes at runtime,
   release builds do not. `SPECS.md` upstream still says otherwise.

For the full grammar-level detail (literals, all keywords, operators,
attributes like `inline`/`naked`/`noreturn`/`section`, `#[target(...)]`,
the two-variable `for i, value in arr` form, etc.) read
`references/spec.md` — it's a condensed rewrite of the language spec and is
the most authoritative reference bundled here.

## Reference files

Read these as needed — don't load all of them for a trivial one-liner, but
for anything nontrivial (multi-function programs, stdlib usage, structs,
error handling, FFI, bare metal) load the relevant one(s) before writing
code:

- `references/syntax.md` — variables, types, functions, control flow,
  structs, enums, optionals/errors, defer, pointers, inline asm, function
  pointers, modules, with short examples for each. Start here for
  "how do I write X in Cinder."
- `references/stdlib.md` — every function/struct exported by the standard
  library (`std/io.cnd`, `std/vec.cnd`, `std/string.cnd`, `std/alloc.cnd`,
  `std/file.cnd`, `std/panic.cnd`, `std/x86.cnd`,
  `std/core/{str,math,mem,ascii,format}.cnd`) with signatures. Check this
  before assuming a helper exists or inventing one — Cinder's stdlib is
  small and deliberately minimal (no generic collections beyond a byte
  Vec/String).
- `references/cli.md` — the `cinder` command-line tool: `check`/`build`/
  `tokens`/`ast`, all flags, build modes, and how to build a bare-metal
  kernel image.
- `references/spec.md` — condensed language specification (grammar-level
  rules, full keyword list, operators, literals, attributes). Consult this
  when uncertain about anything not covered in syntax.md, or to resolve any
  ambiguity.

## Workflow for writing Cinder code

1. Identify what stdlib modules are needed (I/O almost always needs
   `use "std/io.cnd";`) — check `references/stdlib.md` for exact function
   names/signatures rather than guessing C-standard-library names.
2. Write the code following the rules above. Prefer `?T`/`!T` and `else`
   unwrapping over raw pointers. Use pointers only when doing
   FFI, bare-metal, or genuinely need raw memory access.
3. Mentally check: every `let` that's reassigned has `mut`; there is no
   `unsafe` anywhere; every struct/enum name is
   PascalCase and every fn/var is snake_case; every statement ends in `;`.
4. If asked to build/run, use the `cinder` CLI commands in
   `references/cli.md` — don't invent flags.
5. When porting from another language (C, Rust, Python, etc.), don't carry
   over that language's stdlib names, generics, exceptions, or OOP — restate
   the logic using Cinder's actual primitives (byte slices, optionals,
   error-unions, plain structs + free functions, switch on enums).
