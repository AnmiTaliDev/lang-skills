# Cinder Language Spec (condensed)

Condensed from the repo's `SPECS.md` and checked against the compiler source
at commit `a94bfdf`. Where they differ, this file follows the source. This is the
grammar-level reference — consult it for anything `syntax.md` doesn't
cover or when in doubt.

## Design principles

| Principle | Meaning |
|-----------|---------|
| Minimalism | small keyword set, no macros, no templates, no generics |
| Explicitness | code does what it says, no hidden behavior/control flow |
| C compatibility | syntax close to C |
| Bare metal | no runtime, no hidden dependency |
| Safety | static analysis of the whole program; optionals replace NULL |
| Fast compilation | no macros/templates |
| LLVM IR | compiler emits LLVM IR; LLVM does the optimizing |

## Keyword list

```
as and break const continue defer else enum extern fn for if let loop mut
or return switch unsafe use while volatile
```

This is the list from `lib/lexer.rb`. `SPECS.md` omits `and` and `or`.
`unsafe` is deprecated.

Contextual keywords (act as keywords only in the right position, can
still theoretically be identifiers elsewhere): `static`, `struct`,
`export`, `packed`, `sizeof`, `alignof`, `offsetof`, `static_assert`.

Reserved literal spellings: `true`, `false`, `none`, `null`.

## Lexical rules

- Line comment `// ...` to end of line. Block comment `/* ... */`,
  **cannot nest**.
- A `;` is required after every statement. Struct fields / enum variants
  are separated by `;` or `,`; a trailing separator before the closing
  brace is allowed.
- Identifiers: ASCII letters/digits/underscore, first char letter or `_`,
  case-sensitive. `snake_case` for vars/fns, `PascalCase` for
  structs/enums.

## Types table

| Type | Bits | Notes |
|------|------|-------|
| `u8 u16 u32 u64 u128` | as named | unsigned |
| `i8 i16 i32 i64 i128` | as named | signed, two's complement |
| `f32` | 32 | IEEE 754 |
| `f64` | 64 | IEEE 754 |
| `bool` | 8 | true/false |
| `usize` / `isize` | pointer width (64 on x86_64) | unsigned/signed |
| `void` | 0 | function return type only |
| `[N]T` | — | fixed-size array, N is a compile-time constant |
| `[]T` | — | slice: pointer + `.len`; arrays have no `.len` |
| `*T` | — | raw pointer; `*const T`, `*volatile T`, `*const volatile T` qualifiers exist |
| `*void` | — | pointer to unknown type; any `*T` coerces to it implicitly; converting back needs an `as` cast |
| `?T` | — | optional: T or `none` |
| `!T` | — | error union: T or an error (usually an enum variant) |
| `fn(Params) -> Ret` | — | function type/pointer, pointer-sized; `-> Ret` omitted means returns `void` |

## Operators

- Arithmetic: `+ - * / %` (numeric operands; `%` follows dividend's sign)
- Bitwise: `<< >> & \| ^` (integer operands)
- Comparison: `== != < <= > >=` (produce `bool`)
- Logical: `&& \|\| !` (bool operands). `and` is the same as `&&`.
  `or` is the same as `\|\|`, with one extra form: after an `==` comparison,
  a non-bool right operand is compared with the same left value, so
  `c == 32 or 9 or 10` means `c == 32 \|\| c == 9 \|\| c == 10`. This form
  does not work with `\|\|`.
- Assignment: `= += -= *= /= %= <<= >>= &= \|= ^=` (mutable vars only)
- Unary: `-` `!`; address-of `&`; dereference `*`
- Cast: `as` (numeric and pointer casts; unsupported casts are errors)

```rust
let a = 42 as f64;
let reg = 0x40001000 as *u32;
```

- Indexing `arr[i]` / slicing `arr[a..b]`: constant out-of-bounds values are
  not rejected at compile time (`SPECS.md` says they are). Indexes are
  bounds-checked at runtime in debug builds only.
- `if { } else { }` as an expression: both branches must produce the same
  type.
- Struct literal: `Point { x: 1, y: 2 }` (trailing comma OK).
- Array literal: `[1, 2, 3]`; repeat form `[value; count]` (count is a
  non-negative compile-time constant; the two forms — element list vs.
  repeat — cannot mix, e.g. `[1, 2; 3]` is an error).
- Calls pass args in declared order; trailing comma allowed;
  `expr?` on a call result is the error-propagation operator (section 14).
- `sizeof(T)`, `alignof(T)`, `offsetof(T, field)` — all return `usize`,
  are constant expressions, and follow the target's C ABI (LLVM
  DataLayout). `sizeof(struct Pair)` and `sizeof(Pair)` are equivalent.

## Control flow grammar notes

- `for i in a..b` (exclusive) / `for i in a..=b` (inclusive) — loop var
  type is `i32` unless annotated.
- `for value in arr` — iterates arrays/slices/strings by value.
- `for i, value in arr` — **two-variable form**: `i` is `usize` (the
  index), `value` is the element.
- `switch` tests one value against integer literals, integer ranges
  (`a..b`), enum variants (`Enum.Variant` or `.Variant` shorthand), or
  strings; `else` is the catch-all; **no fallthrough between arms**.
- `static_assert(cond);` — `cond` must be a constant expression; valid at
  module top level or inside a function body.

## unsafe (deprecated)

`unsafe { }` and `unsafe fn` are still parsed, have no effect, and produce a
warning. There are no operations that need them. Uninitialized variables are
still an error.

## Packed structs

`packed struct Name { ... }` removes padding and sets alignment to 1.
`export packed struct` is also valid. `packed` applies only to structs. This
feature is in the parser and tests but not in `SPECS.md`.

```rust
packed struct Hdr { tag: u8; size: u32; }
static_assert(sizeof(Hdr) == 5);
static_assert(offsetof(Hdr, size) == 1);
```

## Inline assembly

`asm("template")` or `asm("template" : outputs : inputs)`. Each operand is
`"constraint"(expr)`, operands are comma-separated.

- At most one output. Its constraint starts with `=`, and the expression must
  be a mutable lvalue.
- Read-write (`+`) and memory output (`=m`) operands are not supported.
- Input constraints must not start with `=`.
- There is no clobber list.
- The template uses `$0`, `$1`, numbered from the output, then the inputs.
- With no output, keep both colons separated by a space:
  `asm("nop" : : "r"(v));`
- The value of an `asm` expression with an output is the output value.

```rust
let mut sum: i32 = 0;
asm("movl $1, $0" : "=r"(sum) : "r"(42));
```

## Functions — full attribute list

Attribute keywords: `inline`, `naked`, `noreturn`, `section`. `inline`
and `naked` may appear right after `fn`; all four may appear at the end
of the declaration.

```rust
fn inline add_small(a: u8, b: u8) -> u8 inline { return a + b; }
fn irq_handler() naked { asm("push rdi"); }
fn panic() noreturn { loop { asm("hlt"); } }
fn boot_entry() section(".boot") { }
```

`extern fn` — no body, resolved at link time (libc by default). Calling
one is a plain call. Can be variadic (`...` marks the tail,
fixed params first) and can take function-pointer / `*void` params to
bridge C callbacks (`atexit`, `qsort`, etc.).

## Modules — full rules

- `use "path.cnd";` — path resolves relative to current file, then
  against include dirs (`-I`); `std/` is always searchable; path must
  include the file extension.
- Everything is private by default; `export` makes a declaration public.
  Works on: functions, structs, enums, constants, statics, extern
  functions.
- `#[target("...")]` restricts a declaration to one target. Known
  targets: `x86_64`, `x86_64-freestanding`.

## Build modes & command line

See `references/cli.md` for the full flag reference. Two build modes:
`debug` (default, more runtime checks) and `release`.

## Cinder vs. C — quick contrast (don't default to C idioms!)

| Aspect | C | Cinder |
|--------|---|--------|
| Preprocessor | yes | no |
| Header files | yes | no (modules via `use`/`export`) |
| `defer` | no | yes |
| NULL | yes | `?T` optional types |
| `errno`-style failure | yes | `!T` error types + `?`/`else` |
| Safe strings | no | `[]u8` slices (bounds-known) |
| Raw pointers | yes | yes, no special block needed |
| Macros | yes | no |
| Generics | no | no (Cinder has none either — don't invent them) |

## Glossary

| Term | Meaning |
|------|---------|
| array | fixed-length sequence of values |
| cast | change of a value from one type to another |
| constant | a value computed at compile time |
| enum | a type with named, data-less variants |
| error type | `!T` — a value or an error |
| module | one source file |
| optional type | `?T` — a value or `none` |
| pointer | a value that stores an address |
| slice | a view into memory: a pointer and a length |
| static | a global variable with one instance |
| target | a machine architecture for code generation |
