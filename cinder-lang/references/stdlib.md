# Cinder Standard Library Reference

All modules live under `std/` and are always on the include path — import
with e.g. `use "std/io.cnd";`. The stdlib is intentionally tiny: no
generic collections, no hash maps, no dynamic dispatch beyond function
pointers. `Vec`/`String` are byte-oriented (`u8`) only — there is no
generic `Vec<T>`.

Signatures below were extracted directly from the compiler's bundled
`lib/std/*.cnd` sources, so they are exact.

## std/io.cnd — printing & reading

```rust
export enum PrintType { I32; I64; I128; U8; U64; U128; F64; Bool; S; Ptr; }

export fn putchar(c: i32) -> i32
export fn println(s: []u8)                 // string + newline
export fn print(value: *void, tag: PrintType)  // print(&x, .I32) etc.
export fn print_err(s: []u8)                // stderr
export fn println_err(s: []u8)
export fn print_cstr_err(s: *u8)
export fn read_byte() -> ?u8                // one byte from stdin, none on EOF
export fn read_line(buf: *u8, cap: usize) -> usize

// re-exported libc externs, also usable directly:
export extern fn strlen(s: *u8) -> usize;
export extern fn printf(fmt: *u8, ...) -> i32;
export extern fn sprintf(buf: *u8, fmt: *u8, ...) -> i32;
export extern fn puts(s: *u8) -> i32;
export extern fn fflush(stream: *u8) -> i32;
```

`print`'s tag selects how to interpret `*void`: pass `&value` and the
matching tag (`.I32`, `.U64`, `.F64`, `.S` for a `[]u8` string, `.Bool`,
`.Ptr`, etc.). Example:

```rust
let n = 42;
print(&n, .I32);
let sp: []u8 = " ";
print(&sp, .S);
putchar(10);
```

Use `fflush(null)` after mixing raw `printf` calls with Cinder's buffered
output.

## std/vec.cnd — growable byte buffer

```rust
export struct Vec { data: *u8; len: usize; cap: usize; }

export fn vec_init(v: *Vec)
export fn vec_deinit(v: *Vec)
export fn vec_reserve(v: *Vec, n: usize) -> bool
export fn vec_push(v: *Vec, byte: u8) -> bool
export fn vec_pop(v: *Vec) -> ?u8
export fn vec_get(v: *Vec, i: usize) -> ?u8
export fn vec_set(v: *Vec, i: usize, byte: u8) -> bool
export fn vec_len(v: *Vec) -> usize
export fn vec_as_slice(v: *Vec) -> []u8
export fn vec_clear(v: *Vec)
export fn vec_from_slice(s: []u8) -> Vec
```

Usage pattern (functions take a pointer, not methods):

```rust
let mut v = Vec { data: null, len: 0, cap: 0 };
vec_init(&v);
let ok = vec_push(&v, 10) && vec_push(&v, 20);
for i in 0..vec_len(&v) {
    let b = vec_get(&v, i) else { return 2; };
}
vec_deinit(&v);
```

## std/string.cnd — owning, growable string (built on Vec)

```rust
export struct String { /* wraps a Vec */ }

export fn string_new() -> String
export fn string_from(s: []u8) -> String
export fn string_deinit(s: *String)
export fn string_len(s: *String) -> usize
export fn string_empty(s: *String) -> bool
export fn string_push_byte(s: *String, b: u8) -> bool
export fn string_append(s: *String, t: []u8) -> bool
export fn string_append_str(s: *String, t: *String) -> bool
export fn string_as_slice(s: *String) -> []u8
export fn string_clear(s: *String)
export fn string_equal(a: *String, b: *String) -> bool
export fn string_find(s: *String, needle: []u8) -> ?usize
export fn string_contains(s: *String, needle: []u8) -> bool
```

```rust
let mut s = string_from("cin");
string_push_byte(&s, 'd' as u8);
string_append(&s, "er");
println(string_as_slice(&s));   // "cinder"
string_deinit(&s);
```

## std/core/str.cnd — pure, allocation-free string-slice helpers

```rust
export struct StrSplit { /* .head, .rest — see str_split_once */ }

export fn str_len(s: []u8) -> usize
export fn str_index_of(s: []u8, needle: []u8) -> ?usize
export fn str_contains(s: []u8, needle: []u8) -> bool
export fn str_starts_with(s: []u8, prefix: []u8) -> bool
export fn str_ends_with(s: []u8, suffix: []u8) -> bool
export fn str_trim_start(s: []u8) -> []u8
export fn str_trim_end(s: []u8) -> []u8
export fn str_trim(s: []u8) -> []u8
export fn str_sub(s: []u8, start: usize, end: usize) -> []u8
export fn str_split_once(s: []u8, sep: []u8) -> ?StrSplit
export fn str_compare(a: []u8, b: []u8) -> i32
export fn str_equal(a: []u8, b: []u8) -> bool
export fn str_count(s: []u8, needle: []u8) -> usize
export fn str_parse_u64_radix(s: []u8, radix: u32) -> ?u64
export fn str_parse_u64(s: []u8) -> ?u64
export fn str_parse_i64(s: []u8) -> ?i64
export fn str_parse_f64(s: []u8) -> ?f64
```

```rust
let s = "  cinder  ";
let t = str_trim(s);
if str_starts_with(t, "cin") { println("starts with cin"); }

let n = str_parse_i64("1234") else { return 1; };
```

## std/core/math.cnd

```rust
export fn math_abs(n: i64) -> i64
export fn math_abs_diff(a: i64, b: i64) -> u64
export fn math_min(a: i64, b: i64) -> i64
export fn math_max(a: i64, b: i64) -> i64
export fn math_clamp(n: i64, lo: i64, hi: i64) -> i64
export fn math_min_u(a: u64, b: u64) -> u64
export fn math_max_u(a: u64, b: u64) -> u64
export fn math_clamp_u(n: u64, lo: u64, hi: u64) -> u64
export fn math_is_pow2(n: u64) -> bool
export fn math_log2(n: u64) -> u64
export fn math_pow(n: u64, e: u32) -> u64
export fn math_gcd(a: u64, b: u64) -> u64
export fn math_fabs(n: f64) -> f64
export fn math_floor(n: f64) -> f64
export fn math_ceil(n: f64) -> f64
export fn math_fmin(a: f64, b: f64) -> f64
export fn math_fmax(a: f64, b: f64) -> f64
```

## std/core/mem.cnd

```rust
export fn mem_fill(dst: *u8, value: u8, n: usize)
export fn mem_copy(dst: *u8, src: *u8, n: usize)
export fn mem_move(dst: *u8, src: *u8, n: usize)
export fn mem_zero(dst: *u8, n: usize)
export fn mem_align_up(n: usize, align: usize) -> usize
export fn mem_align_down(n: usize, align: usize) -> usize
```

## std/core/ascii.cnd

```rust
export fn ascii_is_digit(c: u8) -> bool
export fn ascii_is_hex_digit(c: u8) -> bool
export fn ascii_is_alpha(c: u8) -> bool
export fn ascii_is_alnum(c: u8) -> bool
export fn ascii_is_space(c: u8) -> bool
export fn ascii_is_print(c: u8) -> bool
export fn ascii_to_lower(c: u8) -> u8
export fn ascii_to_upper(c: u8) -> u8
export fn ascii_digit_value(c: u8) -> ?u8
```

## std/core/format.cnd — number-to-string into a caller buffer

```rust
export fn fmt_u64(n: u64, buf: *u8, cap: usize) -> usize
export fn fmt_i64(n: i64, buf: *u8, cap: usize) -> usize
export fn fmt_u128(n: u128, buf: *u8, cap: usize) -> usize
export fn fmt_hex_u64(n: u64, buf: *u8, cap: usize) -> usize
export fn fmt_bool(b: bool, buf: *u8, cap: usize) -> usize
export fn fmt_ptr(p: *u8, buf: *u8, cap: usize) -> usize
```

## std/alloc.cnd — heap allocation (libc-backed on hosted targets)

```rust
export fn alloc(n: usize) -> *u8
export fn alloc_realloc(p: *u8, n: usize) -> *u8
export fn dealloc(p: *u8)
```

```rust
let p = alloc(16);
if p == null { return 1; }
unsafe { p[0] = 65; }
dealloc(p);
```

## std/file.cnd — whole-file I/O via the C library

```rust
export fn file_read_all(path: []u8) -> ?[]u8   // heap buffer, NUL-terminated; caller must dealloc(.ptr)
export fn file_write_all(path: []u8, data: []u8) -> bool  // truncates existing file
export fn file_exists(path: []u8) -> bool
export fn file_size(path: []u8) -> ?usize
export fn file_remove(path: []u8) -> bool

// lower-level externs also exposed:
export extern fn fopen(path: *u8, mode: *u8) -> *void;
export extern fn fclose(stream: *void) -> i32;
export extern fn fseek(stream: *void, offset: i64, whence: i32) -> i32;
export extern fn ftell(stream: *void) -> i64;
export extern fn fread(ptr: *u8, size: usize, count: usize, stream: *void) -> usize;
export extern fn fwrite(ptr: *u8, size: usize, count: usize, stream: *void) -> usize;
export extern fn remove(path: *u8) -> i32;
```

```rust
if !file_write_all(path, "hello file") { return 1; }
let data = file_read_all(path) else { return 2; };
println(data);
unsafe { dealloc(data.ptr); }
```

## std/panic.cnd

```rust
export fn panic(msg: []u8) noreturn
export fn panic_cstr(msg: *u8) noreturn
```

## std/x86.cnd — bare-metal port I/O

```rust
export extern fn outb(port: u16, value: u8);
export extern fn outw(port: u16, value: u16);
export extern fn outl(port: u16, value: u32);
export extern fn inb(port: u16) -> u8;
export extern fn inw(port: u16) -> u16;
export extern fn inl(port: u16) -> u32;
```

Only available/meaningful under `--target=x86_64-freestanding`
(`--emit=kernel`). Hosted-only modules (`io`, `alloc`, `vec`, `string`,
`file`) are NOT available on that target since they need libc — see
`cli.md` for bare-metal build flags.
