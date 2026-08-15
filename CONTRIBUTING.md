# Contributing

Thanks for helping teach AI coding agents a new language. This repo works best when each skill is
built the same, verifiable way — please follow this process rather than writing a skill from
memory.

## Ground rules

- **Read the real source, don't guess.** Clone the language's actual compiler/interpreter and
  standard library and derive the rules from there, not from vibes or similarity to a language the
  model already knows. Note in `SKILL.md` where the material came from (repo URL, commit/tag if
  relevant).
- **Call out false-friend traps.** If the language superficially resembles Rust/Zig/C/Go/Python,
  explicitly warn against importing that language's syntax or stdlib names on faith. This is the
  single most common failure mode.
- **Keep `SKILL.md` short.** It should cover: what the language is, a minimal working program, the
  handful of rules that are easy to get wrong, and pointers into `references/` for depth. Don't
  duplicate the full spec in `SKILL.md`.
- **Put depth in `references/` if `SKILL.md` gets too long.** This is optional — a small language
  may fit entirely in `SKILL.md`. If you do split it out, split by concern (`syntax.md`,
  `stdlib.md`, `cli.md`, `spec.md`, or whatever the language needs) so the agent can load only
  what's relevant instead of everything.
- **Verify examples actually compile/run** against the real toolchain before including them.
  A wrong example is worse than no example.

## Adding a new language skill

1. Create a new folder named `<language>-lang/` (or a name that matches an existing convention if
   there's ambiguity).
2. Add a `SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: <language>-lang
   description: >
     Write, edit, review, or explain code in <Language> (.<ext> files, or the
     "<language>" compiler/CLI). Use when the user asks for <Language> code, mentions
     .<ext> files, or asks about its syntax/stdlib/CLI. Don't confuse it with <similar
     languages> just because the syntax looks similar.
   ---
   ```
   The `description` is what the agent uses to decide *when* to invoke the skill, so be specific
   about file extensions, CLI names, and language name variants.
3. If `SKILL.md` alone would get too long, split the depth out into `references/*.md` files
   (syntax, stdlib, CLI, spec are the common ones — add/drop as needed). This step is optional.
4. Update the table in the root [README.md](./README.md).
5. Add yourself to [CONTRIBUTORS.md](./CONTRIBUTORS.md).
6. Open a PR. Mention what source you derived the skill from and how you verified it (e.g. "built
   the compiler from source and ran every code sample against it").

## Improving an existing skill

If you find an agent getting something wrong (hallucinated stdlib function, wrong syntax, importing
rules from a similar language), fix it directly in the relevant `references/*.md` or `SKILL.md`
and explain the mistake you observed in the PR description — that context helps future
contributors avoid regressing the same fix.

## License

By contributing, you agree your contribution is licensed under [CC BY 4.0](./LICENSE), matching
the rest of the repository.
