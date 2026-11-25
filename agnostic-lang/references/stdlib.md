# Agnostic standard library

Based on commit `2617cc7`. `Result` and `Option` need no import. The other
modules do.

`stdio`, `os`, and most of `string` are implemented by the compiler backend, so
the bodies in their `.agn` files are placeholders. `math` and `string.is_empty`
are compiled from their `.agn` bodies. The upstream docs say that every
function is an intrinsic, which is not what the code generator does.

## result

| Type | Fields |
|---|---|
| `Result<T,E>` | `ok bool`, `value T`, `err E` |
| `Option<T>` | `some bool`, `value T` |

Assign a returned `Result` or `Option` to a variable before reading fields.

```agn
var ch Option<int> = string.charAt(s, 0)
if ch.some {
    stdio.Println(ch.value)
}
```

## stdio

| Function | Notes |
|---|---|
| `Print(value int)` | decimal, no newline |
| `Println(value int)` | decimal with newline |
| `PrintStr(text string)` | |
| `PrintlnStr(text string)` | |
| `PrintFloat(value float)` | six digits after the point |
| `PrintlnFloat(value float)` | |
| `PrintBool(value bool)` | `true` or `false` |
| `PrintlnBool(value bool)` | |
| `PrintChar(ch int)` | one byte |
| `ReadInt() int` | |
| `ReadChar() int` | one byte |
| `ReadLine(buffer *u8, maxlen int) int` | returns the byte count |
| `Flush()` | no-op, output is unbuffered |

`Println` takes only integers. There is no `Printf`. Use a template string with
`PrintlnStr` for mixed output.

## math

All arguments are `int`.

| Function | Notes |
|---|---|
| `Max(a, b) int` | |
| `Min(a, b) int` | |
| `Pow(base, exp) int` | non-negative `exp` |
| `Sqrt(n) int` | integer square root, positive input |
| `GCD(a, b) int` | positive input |
| `LCM(a, b) int` | positive input |
| `Fact(n) int` | factorial |
| `IsEven(n) bool` | |
| `IsOdd(n) bool` | |
| `IsPrime(n) bool` | |
| `Sign(x) int` | 1 if `x > 0`, else 0. Negative input also gives 0 |
| `Clamp(value, min, max) int` | |
| `SumRange(n) int` | `n * (n + 1) / 2` |
| `Fib(n) int` | |

There are no float math functions.

`IsPrime`, `IsEven`, and `IsOdd` return `bool`. Write `if !math.IsPrime(n)`.
The upstream control flow page still compares the result with `0`.

## string

| Function | Notes |
|---|---|
| `len(s) int` | byte length |
| `compare(s1, s2) int` | -1, 0, or 1 |
| `concat(s1, s2) string` | |
| `is_empty(s) bool` | |
| `indexOf(s, sub) Option<int>` | first occurrence |
| `contains(s, sub) bool` | |
| `startsWith(s, prefix) bool` | |
| `endsWith(s, suffix) bool` | |
| `charAt(s, index) Option<int>` | `some=false` if out of bounds |
| `substr(s, start, len) string` | clamped to bounds |
| `toUpper(s) string` | ASCII only |
| `toLower(s) string` | ASCII only |

There is no split, trim, replace, or string to number parsing.

## os

| Function | Notes |
|---|---|
| `ArgCount() int` | includes argv[0] |
| `Arg(index int) string` | |
| `OpenRead(path string) Option<int>` | value is a file descriptor |
| `OpenCreate(path string) Option<int>` | create or truncate for writing |
| `Close(fd int) int` | |
| `ReadFd(fd int, buffer *u8, maxlen int) Option<int>` | value is bytes read |
| `WriteFd(fd int, data string) Option<int>` | value is bytes written |
| `Exit(code int)` | exits immediately |

```agn
var rfd Option<int> = os.OpenRead(path)
if !rfd.some {
    os.Exit(1)
}
var buf [64]u8
var n Option<int> = os.ReadFd(rfd.value, &buf, 63)
os.Close(rfd.value)
```

To use a byte buffer as a string, make sure it ends with a zero byte, then cast
with `&buf as string`.
