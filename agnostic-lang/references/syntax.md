# Agnostic syntax

Based on commit `2617cc7`. Examples are taken or adapted from the upstream
`examples/` and `docs/`. Many are fragments.

## Contents

- [Source files](#source-files)
- [Types](#types)
- [Variables](#variables)
- [Conversions](#conversions)
- [Operators](#operators)
- [Strings](#strings)
- [Functions and closures](#functions-and-closures)
- [Control flow](#control-flow)
- [Arrays and pointers](#arrays-and-pointers)
- [Structs and generics](#structs-and-generics)
- [Comptime](#comptime)
- [Modules](#modules)

## Source files

Order: one `package` line, then all `import` lines, then `struct` and `func`
declarations. Extension: `.agn`.

Line comments start with `//` or `#`. Block comments do not exist.

Keyword aliases: `fn` = `func`, `let` = `var`, `use` = `import`, `while` and
`loop` = `for`. `let` does not make a binding immutable. Prefer the primary
names.

## Types

| Type | Notes |
|---|---|
| `i8` `i32` `i64` | signed integers |
| `u8` `u32` `u64` | unsigned integers |
| `int` | alias of `i64` |
| `f64`, `float` | the only float type |
| `bool` | |
| `string` | |
| `*T` | pointer |
| `func(T1, T2) -> R` | function value; omit `-> R` for no result |

There are no 16-bit types, no `f32`, no `char`, no `usize`.

Integer literals are decimal only. No hex, binary, octal, `_` separators, or
character literals. `3.14` is a float literal.

## Variables

```agn
var x int = 5
var y = 5
var z int
```

A colon (`var x: int = 5`) is optional. Every variable is mutable. A variable
declared in a block stays visible until the end of the enclosing function, so
use distinct names.

Assignment forms:

```agn
x = 6
arr[0] = 6
point.x = 6
*ptr = 6
```

The target is a single name with at most one index or one field. `a.b.c = v`,
`arr[i].x = v`, and `*arr[i] = v` are not supported. There are no compound
assignments: write `i = i + 1`.

## Conversions

Implicit conversion happens only when no value can be lost:

- `i8` to `i32` to `i64`, `u8` to `u32` to `u64`
- an unsigned type to a wider signed type
- `i8`, `i32`, `u8`, `u32` to `f64`
- an integer literal (or negated literal) to a numeric type that holds it
- `*[N]T` to `*T`
- `*u8`, `*i8`, `*[N]u8`, `*[N]i8` to `string`

Everything else needs `as`. Narrowing, signedness changes, float to integer,
and `i64` or `u64` to `f64` all need it.

`as` also allows: `bool` to integer, between pointers and `string`, and between
a pointer and `i64` or `u64`. There is no cast to `bool`. A `string` does not
cast to a number.

`as` binds tighter than binary operators and looser than unary ones:
`a + b as i64` is `a + (b as i64)`.

Both operands of a binary operator must reach one type. A literal takes the
other operand's type. `i32` with `u32`, or `f64` with `i64`, is an error.

## Operators

`+ - * / %`, `== != < <= > >=`, `&& || !`, `& | ^ << >>`, unary `-`,
`&` (address), `*` (dereference), `as`, `++` (string concatenation).

There is no `~` and no ternary operator. `%`, bitwise operators, and shifts do
not accept floats.

`&&` and `||` evaluate both sides in the `llvm` backend:

```agn
var ok bool = false
if i < 8 {
    if arr[i] > 0 {
        ok = true
    }
}
```

The parser does not chain all precedence levels. In particular:

- the right side of a comparison (`<`, `<=`, `>`, `>=`) cannot be a bare shift:
  write `a < (b << 1)`
- the right side of `&&` cannot be a bare `&`, `|`, or `^` expression: write
  `a && ((x & 1) == 1)`
- `==` binds tighter than `&`, `^`, `|`: write `(x & 1) == 0`

When in doubt, add parentheses.

## Strings

Escapes: `\n`, `\t`, `\r`, `\\`, `\"`.

```agn
var greeting string = "foo" ++ "bar"
```

A string literal with `$(expr)` is a template string, evaluated at runtime. A
format may follow a colon: optional `0`, optional width, then `d`, `x`, `X`,
or `s`.

```agn
stdio.PrintlnStr("hello $(name), value=$(n:05d)")
```

The expression inside `$(...)` cannot contain a string literal, because the
quote ends the outer string.

Comparison: `==` and `!=` compare addresses. Compare contents with
`string.compare(a, b) == 0`.

Byte access: `string.charAt(s, i)`. A literal such as `"hello"[0]` can be
indexed and gives a `u8`.

## Functions and closures

```agn
func add(a int, b int) int {
    return a + b
}
```

The return type follows the parameters. `->` before it is optional. Omit the
type for no result. There is one return value at most. There is no overloading,
default argument, or variadic function. Declaration order does not matter.

A function that ends without `return` yields the zero value of its return type.

Function values and closures:

```agn
func makeCounter() func() -> i64 {
    var count i64 = 0
    var next func() -> i64 = func() -> i64 {
        count = count + 1
        return count
    }
    return next
}

func apply(f func(i64) -> i64, x i64) i64 {
    return f(x)
}
```

Closures capture by reference. A named function can be used as a value by
writing its name without parentheses. A function value is called through a
plain name, or through a struct field (`platform.write(1, 0, 4)`).

## Control flow

```agn
if x % 15 == 0 {
    stdio.PrintlnStr("FizzBuzz")
} else {
    if x % 3 == 0 {
        stdio.PrintlnStr("Fizz")
    } else {
        stdio.Println(x)
    }
}
```

Braces are required. `else if` is a parse error: nest the `if` inside the
`else` block.

```agn
var i int = 0
for i < 10 {
    i = i + 1
}

for {
    break
}
```

`break` and `continue` work only inside a loop in the same function or closure.
There is no `switch` or `match`.

## Arrays and pointers

```agn
var arr [8]int
arr[0] = 5
var first int = arr[0]
```

The size is an integer literal. The element type is a single type name, so
arrays of pointers, arrays of arrays, and arrays of generic types cannot be
written. There is no length property. Initialize elements one by one.

```agn
func set(p *int, v int) {
    *p = v
}
```

Take the address of a plain variable only: `&x`, `&buf`. Do not write
`&arr[i]` or `&s.field`: the type checker accepts them, but the `llvm` backend
produces the value instead of its address. Pointers cannot be indexed and have
no arithmetic. For address arithmetic, cast to `u64` and back.

Field access through a pointer does not work: `p.x` with `p *Point` is a type
error. Use a method, or copy with `var v Point = *p`.

## Structs and generics

```agn
struct Vector2 {
    x i64,
    y i64
}

func (v Vector2) Length2() i64 {
    return v.x * v.x + v.y * v.y
}

func (v Vector2) SetX(newX i64) {
    v.x = newX
}
```

```agn
var a Vector2 = Vector2{x: 3, y: 4}
a.SetX(99)
var n i64 = a.Length2()
```

Fields are separated by a comma or a newline. Literals must use field names.
Name every field.

The receiver type is written without `*`. The receiver is passed by reference,
so a method that assigns to a field changes the caller's variable.

There is no inheritance, embedding, or interface. `pub struct` is a parse
error.

Generic structs use `<...>`:

```agn
struct Pair<A, B> {
    first A
    second B
}

var p Pair<int,string> = Pair<int,string>{first: 1, second: "one"}
```

`Result<T,E>` (`ok`, `value`, `err`) and `Option<T>` (`some`, `value`) are
always available. There are no `Ok`, `Err`, or `Some` constructors and no `?`
operator.

```agn
func safeDivide(a int, b int) Result<int,string> {
    if b == 0 {
        return Result<int,string>{ok: false, value: 0, err: "divide by zero"}
    }
    return Result<int,string>{ok: true, value: a / b, err: ""}
}
```

Generic functions use `comptime` parameters, not `<...>`. Type arguments are
passed explicitly.

```agn
func max(comptime T type, a T, b T) T {
    if a > b {
        return a
    }
    return b
}

func addN(comptime n int, x int) int {
    return x + n
}
```

```agn
var m int = max(int, 3, 7)
var s int = addN(5, 10)
```

Not supported: `comptime` parameters on methods, `comptime` array sizes, and
type argument inference.

## Comptime

```agn
comptime {
    var sum int = 0
    var i int = 0
    for i < 5 {
        sum = sum + i
        i = i + 1
    }
    var total int = sum
}
stdio.Println(total)
```

Names computed in a `comptime` block remain usable until the end of the
function. A statement the evaluator cannot run is left as runtime code without
an error. The limit is 10,000,000 evaluated statements.

An `if` directly inside `comptime` is conditional compilation: only the taken
branch is kept, and the condition must be evaluable at compile time.

```agn
comptime {
    if TARGET_OS == "linux" {
        platform.write = linuxWrite
    } else {
        platform.write = stubWrite
    }
}
```

Constants: `TARGET_OS` (`linux`, `freebsd`, `windows`, `hurd`), `TARGET_ARCH`
(`x86_64`), `MEM_MODE` (`arc`, `manual`, `orc`).

## Modules

`import "name"` loads `name.agn`. Search order: the directory of the main
source file, `stdlib/` next to the compiler binary,
`../share/agnostic/stdlib/` relative to it, then the build-time source
`stdlib/`.

A function is exported if it has `pub` **or its name starts with an uppercase
letter**. The upstream docs mention only `pub`. Other functions in a module are
not type-checked or compiled, so exported functions cannot call them.

```agn
package shapes

pub fn Square(x int) int {
    return x * x
}
```

Call it as `shapes.Square(4)`. Inside the same module, an exported function can
call another exported one by its bare name.

Only functions are loaded from a module. Structs declared there are not
available. Import cycles are not diagnosed.
