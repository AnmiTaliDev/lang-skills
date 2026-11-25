# Agnostic compiler

Based on commit `2617cc7`.

## Usage

```
agnostic <source.agn> [options]
```

| Option | Meaning |
|---|---|
| `--backend=llvm\|gcc` | backend, default `llvm`. `--llvm` and `--gcc` are shorthands |
| `--mem=arc\|manual\|orc` | memory mode, default `arc` |
| `--target-os=linux\|freebsd\|windows\|hurd` | default `linux` |
| `--output=<path>` | output path, default is the source path without `.agn` |
| `-c`, `--compile-only` | write `<output>.o` (`.obj` for windows) and do not link |
| `--version` | print the version |
| `-h`, `--help` | print usage |

Use the `--output=<path>` form. There is no `-o`. There are no `build`, `run`,
or `check` subcommands.

Linking uses `cc` on the host (`lld-link` for `windows`). Binaries are static
and do not link libc.

## Memory modes

- `arc`: reference counting.
- `manual`: allocations are never freed automatically.
- `orc`: region based. Allocations are freed when the function that owns the
  region returns. Functions whose results or arguments can carry a string,
  pointer, or function allocate in the caller's region. A loop in `main` that
  creates strings through such a function keeps them until `main` exits.

An allocation failure prints `fatal error: out of memory` and exits with code 2.

## Targets

Only `x86_64`.

- `windows`: produces `<output>.exe`, needs `--backend=llvm` and `lld-link`.
- `hurd`: upstream states it has not been run on a Hurd system.

## Building the compiler

Needs CMake 3.20 or later, a C++20 compiler, and LLVM development files. The
`gcc` backend needs libgccjit. The `windows` target needs `clang`, `llvm-lib`,
and `llvm-dlltool` at build time.

```sh
cmake -S . -B build
cmake --build build -j$(nproc)
./build/src/cli/agnostic --version
```

The build first makes a stage 0 compiler with the C++ lexer, then compiles the
Agnostic lexer in `src/lexer/lexer.agn` with it.

## Diagnostics

Errors show the message, `file:line:column`, and the source line, sometimes
with a "did you mean" suggestion. Locations are reported against the main source
file, including type errors that come from an imported module.
