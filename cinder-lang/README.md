# cinder-lang

An agent skill that teaches AI coding agents (Claude Code and others that support the [agent
skills format](https://docs.claude.com/en/docs/claude-code/skills)) how to write, edit, review,
and explain code in [Cinder](https://github.com/z3nnix/cinder), a minimalist,
C-like systems language that compiles to LLVM IR.

The skill was written by cloning and reading the real Cinder compiler and standard library source,
so the agent works from verified rules instead of guessing syntax by analogy to Rust/Zig/C.

## What's included

- `SKILL.md` — entry point: quick orientation, core rules, minimal working program, workflow.
- `references/syntax.md` — variables, types, control flow, structs, enums, optionals/errors,
  pointers, inline asm, modules.
- `references/stdlib.md` — every function/struct exported by Cinder's standard library.
- `references/cli.md` — the `cinder` CLI (`check`/`build`/`tokens`/`ast`, flags, bare-metal builds).
- `references/spec.md` — condensed, authoritative language spec.

## Installing

With [`npx skills`](https://www.npmjs.com/package/skills):

```bash
npx skills add AnmiTaliDev/lang-skills --skill cinder-lang
```

Or copy this folder directly into your agent's skills directory (e.g. `.claude/skills/` for
Claude Code).

## Using it

Once installed, just ask your agent to write, port, or explain Cinder code (`.cnd` files, or
anything mentioning the `cinder` compiler) — the skill loads automatically when relevant.

## License

[CC BY 4.0](../LICENSE), like the rest of this repository.
