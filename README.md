# lang-skills

A collection of agent skills that teach AI coding agents how to write correct code in niche,
unpopular, or brand-new programming languages that a model doesn't already know well from its
training data. These skills follow the [agent skills
format](https://docs.claude.com/en/docs/claude-code/skills) and work with any AI coding agent that
supports it (Claude Code and others), not just Claude.

Each skill is a self-contained folder with a `SKILL.md` (the entry point the agent loads),
optionally backed by `references/` files with deeper grammar, stdlib, and CLI details. The idea is
simple: instead of letting the model guess at syntax by analogy to Rust/C/Zig/whatever it
superficially resembles, give it a verified, condensed spec to work from.

## Available skills

| Skill | Language | Description |
|---|---|---|
| [`cinder-lang`](./cinder-lang) | [Cinder](https://github.com/z3nnix/cinder) | A minimalist, safe-by-default, C-like systems language compiling to LLVM IR. |

## Installing a skill

Each skill can be installed individually with [`npx skills`](https://www.npmjs.com/package/skills):

```bash
npx skills add AnmiTaliDev/lang-skills --skill cinder-lang
```

Swap `cinder-lang` for the folder name of whichever skill you want from the table above.

## How a skill is structured

At minimum, a skill is just:

```
<language>-lang/
└── SKILL.md              # entry point: quick orientation, core rules, workflow
```

For languages with enough surface area that `SKILL.md` would get too long, split the depth out
into an optional `references/` folder:

```
<language>-lang/
├── SKILL.md              # entry point: quick orientation, core rules, workflow
└── references/            # optional — only if SKILL.md alone isn't enough
    ├── syntax.md          # variables, types, control flow, structs, etc.
    ├── stdlib.md           # standard library reference
    ├── cli.md              # compiler/CLI usage
    └── spec.md             # condensed, authoritative language spec
```

If you do add `references/`, `SKILL.md` should stay short enough to always load, and should point
to the right `references/*.md` file for anything nontrivial rather than cramming everything
into one file.

## Adding a new skill

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## Contributors

See [CONTRIBUTORS.md](./CONTRIBUTORS.md).

## License

All skills in this repository are licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) unless a skill's own folder states
otherwise. See [LICENSE](./LICENSE).
