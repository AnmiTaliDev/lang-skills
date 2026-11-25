# Cinder Syntax Reference

Examples are adapted from the `docs/` chapter files and compiler tests in the
Cinder repo (commit `a94bfdf`). Many are fragments, not whole programs.

## Table of contents
- [Comments](#comments)
- [Variables (let/mut/const/static)](#variables)
- [Types & literals](#types--literals)
- [Functions](#functions)
- [Control flow](#control-flow)
- [Structs](#structs)
- [Enums](#enums)
- [Arrays & slices](#arrays--slices)
- [Optionals & errors](#optionals--errors)
- [defer](#defer)
- [Pointers & inline asm](#pointers--inline-asm)
- [Function pointers](#function-pointers)
- [FFI (extern fn)](#ffi-extern-fn)
- [Modules (use/export)](#modules)

---

## Comments

```rust
// line comment

/*
   block comment (cannot nest)
*/
```

---

## Variables

`let` infers the type from the initializer; a colon adds an explicit
annotation. Bindings are **immutable by default** — add `mut` to allow
reassignment.

```rust
let x = 42;            // i32
let name = "cinder";   // []u8
let pi = 3.14;          // f64
let a: u8 = 200;
let mut counter = 0;
counter += 1;
```

Shadowing is allowed: a new `let` with the same name in the same scope
hides the old binding.

```rust
let x = 1;
let x = x + 10;   // new x = 11
```

`const` is a compile-time constant (value must be computable at compile
time):

```rust
const MAX_ITEMS: u32 = 64;
const BLOCK_SIZE = 4096;
```

`static` is a single global instance for the whole program:

```rust
static CALLS: u32 = 0;

fn ping() {
    CALLS += 1;
}
```

A `static` can be placed in a named linker section (bare-metal use):

```rust
static MB_HEADER: [3]u32 = [0x1BADB002, 0, 0xE4524FFE] section(".multiboot");
```

---

## Types & literals

### Integers

| Type | Notes |
|------|-------|
| `u8 i8 u16 i16 u32 i32 u64 i64 u128 i128` | fixed width |
| `usize isize` | pointer-width (64-bit on x86_64) |

Default integer literal type is `i32`. Suffix to change type; `_` is a
digit separator. Bases: none=decimal, `0x`=hex, `0b`=binary, `0o`=octal.

```rust
let a = 42;          // i32
let b = 42u32;        // u32
let c = 0xFF;          // i32 = 255
let d = 0b1010;        // i32 = 10
let e = 0o755;         // i32 = 493
let f = 1_000_000;
let g = 255u8;
```

A literal must fit its target type at compile time
(`let h: u8 = 300;` is a compile error; so is `let h: u8 = -1;` since unary
`-` on an unsigned type doesn't fit).

### Floats

Default float literal type is `f64`; suffix `f32`/`f64` to change it.

```rust
let a = 3.14;    // f64
let b = 3.14f32; // f32
```

### bool

```rust
let ok = true;
let done = false;
if ok && !done { ... }
```

`and` and `or` are also accepted. After `==`, `or` can chain values:
`if c == 32 or 9 or 10 { ... }` compares `c` with each value.

### char

A char literal is one ASCII byte in single quotes, **type `i32`**.
Escapes: `\n \t \r \0 \\ \' \" \xNN`.

```rust
let letter = 'A';   // 65
let esc = '\x1b';   // 27
```

### Strings

A string literal has type `[]u8` (a byte slice), not a null-terminated
C string, unless you use the `c` prefix.

| Prefix | Meaning |
|--------|---------|
| none | escapes processed, type `[]u8` |
| `r"..."` | raw, escapes kept literal, type `[]u8` |
| `b"..."` | byte string, escapes kept literal, type `[]u8` |
| `c"..."` | null-terminated, type `*u8` (for C interop) |

```rust
let s = "Hello\nWorld";      // []u8
let raw = r"C:\temp";         // []u8, backslash kept
let cstr = c"nul-terminated"; // *u8
```

Indexing a string/slice returns a `u8` byte: `s[0]`. Iterating (`for b in
s`) visits bytes.

### Other primitive types

`void` — only valid as a function return type (meaning "no value").

---

## Functions

```rust
fn add(a: i32, b: i32) -> i32 {
    return a + b;
}

fn log(msg: []u8) {
    // no return type => void function
}
```

The **last expression in the body is an implicit return** if it has no
trailing semicolon and matches the return type:

```rust
fn square(a: i32) -> i32 {
    a * a   // implicit return, no `return` keyword needed
}
```

`return` still works for early exits:

```rust
fn classify(v: i32) -> []u8 {
    if v < 0 { return "negative"; }
    if v == 0 { return "zero"; }
    return "positive";
}
```

### Function attributes

Attribute keywords: `inline`, `naked`, `noreturn`, `section`. `inline` and
`naked` may also appear right after `fn`; all four may appear at the end
of the declaration:

```rust
fn inline add_small(a: u8, b: u8) -> u8 inline {
    return a + b;
}

fn irq_handler() naked {
    asm("push rdi");
}

fn panic() noreturn {
    loop { asm("hlt"); }
}

fn boot_entry() section(".boot") {
    // body
}
```

### main

```rust
fn main() -> i32 { return 0; }     // exit code = return value
fn main() { }                       // void main => exit code 0
fn main(argc: i32, argv: **u8) -> i32 { ... }  // C-style argv
```

Parameters are **read-only values** (pass-by-value); pass a pointer
(`*T`) if the callee needs to mutate the caller's data.

---

## Control flow

### if / else (statement and expression)

```rust
if temp > 30 {
    println("hot");
} else if temp > 20 {
    println("warm");
} else {
    println("cold");
}

// if as an expression — both branches must have the same type
let label = if n % 2 == 0 { "even" } else { "odd" };
```

### loop / break / continue

```rust
let mut i = 0;
loop {
    i += 1;
    if i % 2 == 0 { continue; }
    if i > 7 { break; }
    // use i
}
```

### while

```rust
let mut n = 1;
while n < 100 {
    n *= 2;
}
```

### for over ranges

`a..b` is exclusive of `b`; `a..=b` is inclusive.

```rust
let mut sum = 0;
for i in 0..=10 {   // 0..10 inclusive
    sum += i;
}
```

A `usize` range is idiomatic for indexing:

```rust
for i in 0usize..s.len { ... }
```

### for over collections

Iterates arrays, slices, and strings by value:

```rust
let primes = [2, 3, 5, 7];
let mut total = 0;
for p in primes {
    total += p;
}
```

Two-variable form gives index (`usize`) and value:

```rust
for i, value in arr {
    // i: usize, value: element type
}
```

### switch

Matches integers, integer ranges, enum variants, or strings. No
fallthrough — each arm is a self-contained alternative. `else` is the
catch-all:

```rust
let code = 2;
switch code {
    0 => println("zero");
    1..3 => println("small");
    else => println("large");
}
```

Switching on an enum, with the `.Variant` shorthand once the type is known:

```rust
enum Direction { North; East; South; West; }

fn arrow(d: Direction) -> []u8 {
    switch d {
        .North => { return "^"; }
        .East => { return ">"; }
        .South => { return "v"; }
        .West => { return "<"; }
    }
}
```

### static_assert

Compile-time assertion; fails the build if false. Valid at module top level
or inside a function body.

```rust
static_assert(sizeof(Pair) == 8);
static_assert(4 * 4 == 16);
```

---

## Structs

```rust
struct Point {
    x: i32;
    y: i32;
}

let p = Point { x: 3, y: 4 };
let x = p.x;
```

Mutating a field requires a `mut` binding, and you assign the field, not
the struct:

```rust
let mut c = Counter { value: 0, step: 2 };
c.value += c.step;
```

**No methods.** The idiomatic "method" pattern is a free function taking a
pointer to the struct; field access through the pointer is automatic
(`r.w` reads through `r: *Rect`):

```rust
struct Rect { w: i32; h: i32; }

fn area(r: *Rect) -> i32 {
    return r.w * r.h;
}

fn grow(r: *Rect, k: i32) {
    r.w *= k;
    r.h *= k;
}
```

Structs cannot be compared with `==` (no aggregate comparison).

`packed struct` has no padding and has alignment 1:

```rust
packed struct Hdr { tag: u8; size: u32; }
static_assert(sizeof(Hdr) == 5);
```

`sizeof(T)`, `alignof(T)`, `offsetof(T, field)` are compile-time constants
matching the target's C ABI:

```rust
struct Pair { a: i32; b: u8; }
static_assert(sizeof(Pair) == 8);
static_assert(offsetof(Pair, b) == 4);
```

---

## Enums

Plain, data-less variants (no tagged-union payloads):

```rust
enum Color { Red; Green; Blue; }

fn name(c: Color) -> []u8 {
    switch c {
        Color.Red => { return "red"; }
        Color.Green => { return "green"; }
        Color.Blue => { return "blue"; }
    }
}
```

`.Variant` shorthand once the type is known/inferred:

```rust
let c: Color = .Red;
```

Enums are the idiomatic error type for `!T` functions:

```rust
enum IoError { NotFound; PermissionDenied; DeviceBusy; }
```

An enum value is stored as its variant index. Convert it with a cast:
`let idx = Color.Blue as i32;` (2).

---

## Arrays & slices

`[N]T` is a fixed-size array (N is a compile-time constant); `[]T` is a
slice (pointer + length, has a `.len` field; arrays don't).

```rust
let primes: [4]i32 = [2, 3, 5, 7];
let zeroes = [0; 8];          // repeat literal: eight zeroes

let arr = [10, 20, 30, 40, 50];
let s = arr[1..4];             // slice [20, 30, 40]
let len = s.len;
let whole = arr[..];           // convert whole array to a slice
```

Slice operator forms: `[..]`, `[a..]`, `[..b]`, `[a..b]`.

Constant out-of-bounds indexes are not rejected at compile time. Indexes are
bounds-checked at runtime in debug builds only.

Strings are `[]u8` slices too — same indexing/slicing rules, `.ptr` /
`.len` fields.

```rust
fn total(vals: []i32) -> i32 {
    let mut t = 0;
    for v in vals { t += v; }
    return t;
}
```

---

## Optionals & errors

Do not use `null` to signal failure. `?T` = value or `none`. `!T` = value or an
error (usually an enum variant).

```rust
fn maybe_name(id: i32) -> ?[]u8 {
    if id == 0 { return none; }
    return "cinder";
}
```

**`else` unwraps** — the block runs only if the value was `none`/error, and
must leave the function (return / break / panic):

```rust
let x = maybe else { println("was none"); return 0; };
// x is now the unwrapped value
```

Error functions use `!T` as the return type (no `->` needed before `!T`):

```rust
enum Err { DivisionByZero; }

fn safe_div(a: i32, b: i32) !i32 {
    if b == 0 { return Err.DivisionByZero; }
    return a / b;
}

let r = safe_div(10, 2) else {
    println("error");
    return 1;
};
```

**`?` propagates an error** — only valid inside a function that itself
returns `!T`:

```rust
fn checked(a: i32, b: i32) !i32 {
    if b == 0 { return Err.DivideByZero; }
    return a / b;
}

fn half(a: i32) !i32 {
    let q = checked(a, 2)?;   // propagates Err up if checked() failed
    return q;
}
```

Stdlib lookups/parsers commonly return `?T`:
`read_byte() -> ?u8`, `vec_get() -> ?u8`, `str_index_of() -> ?usize`,
`str_parse_i64(s) -> ?i64`.

---

## defer

Runs a statement at the end of the enclosing scope, even on early `return`.
Multiple `defer`s run **LIFO** (last defer runs first):

```rust
fn main() -> i32 {
    defer { println("cleanup (runs first is actually the LAST defer)"); }
    println("doing work");
    return 0;
}
```

A conditional defer, evaluated at scope-exit time:

```rust
defer if flag != 0 { println("flagged"); };
```

Idiomatic for resource cleanup: open in one place, `defer` the close/free
right after.

---

## Pointers & inline asm

Address-of, dereference, pointer arithmetic, pointer indexing and slicing,
pointer casts, and `asm` are allowed everywhere. Do not wrap them in
`unsafe`: it is deprecated and only produces a warning.

```rust
let mut x = 5;
let p = &x;
*p += 10;
```

"Method" / mutate-through-pointer pattern:

```rust
fn swap(a: *i32, b: *i32) {
    let t = *a;
    *a = *b;
    *b = t;
}
```

`null` is the null pointer literal. Never dereference it:

```rust
let p: *i32 = null;
if p == null { println("null pointer"); }
```

Pointer qualifiers:

```rust
let a: *const u32 = &value;
let b: *volatile u32 = &reg;
let c: *const volatile u32 = &reg;
```

`*void` is a pointer to an unknown type. Any pointer coerces to `*void`
implicitly. Cast it back with `as`. It cannot be dereferenced, indexed, or
sliced:

```rust
let opaque: *void = &n;
let back: *u64 = opaque as *u64;
```

Pointer arithmetic:

```rust
let base = arr[..].ptr;
let v1 = *base;
let v2 = *(base + 1);
```

Inline assembly. Operand rules are in `spec.md`:

```rust
asm("wfi");

let mut sum: i32 = 0;
asm("movl $1, $0" : "=r"(sum) : "r"(42));
```

---

## Function pointers

Type syntax: `fn(Params) -> Ret`. A bare function name (or `&name`) is a
pointer-sized value:

```rust
fn double(v: i32) -> i32 { return v * 2; }

fn apply(f: fn(i32) -> i32, v: i32) -> i32 {
    return f(v);
}

let f: fn(i32) -> i32 = double;
apply(f, 5);
apply(triple, 5);   // bare function name works directly too
```

`null` is assignable to / comparable with function-pointer types:

```rust
let f: fn() = null;
if f == null { println("not set"); }
```

Struct-of-function-pointers = tiny vtable / dispatch table:

```rust
struct Format {
    header: fn() -> []u8;
    body: fn(i32) -> []u8;
}

fn emit(fmt: *Format, v: i32) {
    println(fmt.header());
    println(fmt.body(v));
}
```

---

## FFI (extern fn)

`extern fn` declares a function with no body, resolved at link time
(links against libc by default). It is called like any other function:

```rust
extern fn abs(n: i32) -> i32;
let v = abs(-42);
```

Variadic externs use `...` for the tail (this is how the stdlib declares
`printf`):

```rust
extern fn printf(fmt: *u8, ...) -> i32;
```

`c"..."` literals are null-terminated `*u8`, the shape C expects:

```rust
extern fn my_strlen(s: *u8) -> usize;
let msg = c"hello, libc!";
let len = my_strlen(msg);
```

C allocations come back as raw pointers and can be dereferenced and indexed
directly:

```rust
extern fn malloc(size: usize) -> *u8;
extern fn free(p: *u8);

let p = malloc(64);
p[0] = 65;
free(p);
```

C function pointer callbacks:

```rust
extern fn atexit(cb: fn() -> void) -> i32;
extern fn qsort(base: *void, nmemb: usize, size: usize,
                compar: fn(*void, *void) -> i32);
```

Struct layout matches the C ABI (sizes/alignments/offsets follow the
target's LLVM DataLayout) — use `sizeof`/`offsetof` to stay in sync with a
C struct definition, e.g. mirroring `struct timespec`.

---

## Modules

`use "path.cnd";` imports a module; the path resolves relative to the
current file, then against include dirs (`std/` is always searchable).
Everything in a module is **private unless marked `export`**.

```rust
// lib/queue.cnd
export struct Queue { data: [16]u8; head: u8; tail: u8; }

export fn q_init() -> Queue {
    return Queue { data: [0; 16], head: 0, tail: 0 };
}

fn q_secret() { }   // private — not visible to importers
```

```rust
// main.cnd
use "queue.cnd";
use "std/io.cnd";

fn main() -> i32 {
    let mut q = q_init();
    return 0;
}
```

Build with an extra include dir: `cinder build main.cnd --emit=bin -o main -I lib`.

`export` works on: functions, structs, enums, constants, statics, extern
functions.

```rust
export const VERSION = 1;
export static SEED: u32 = 0x1234;
export extern fn puts(s: []u8) -> i32;
```

`#[target("...")]` restricts a declaration to one target
(`x86_64` or `x86_64-freestanding`):

```rust
#[target("x86_64-freestanding")]
fn uart_putc(c: u8) { }
```
