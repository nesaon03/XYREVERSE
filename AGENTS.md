# Agent guide

How to use this pack.

## Load a skill by trigger

Each `SKILL.md` opens with YAML frontmatter:

```yaml
---
name: ida-pro-advanced-re
description: Use when doing static binary analysis in IDA Pro. Headless automation, IDAPython, Microcode deobfuscation, type recovery.
---
```

The `description` is the trigger. Match the user's task against it and load the file before starting — the pitfalls list is the part that saves time, and it is written from failures, not from documentation.

Load **at most one or two** skills for a task. Loading all nine wastes context and buries the relevant procedure.

## What a skill gives you

1. **Toolchain** — the tools that matter and the version boundaries that break.
2. **Working commands** — copy-pasteable invocations, not paraphrases.
3. **Core procedure** — the shortest correct path through the task.
4. **Pitfalls** — the failure modes. Read this section even if you skip the rest.

## Output discipline

- Cite `file:line` or an address for every claim about a binary. No claim from memory.
- A constant that a tool must consume goes in verbatim. Never retype an offset, address, IOCTL code, or symbol from an earlier turn.
- Distinguish *verified* (you ran it, you read the bytes) from *assumed* (it follows from the layout). Label which.
- When a check is deterministic — a hash, an exit code, a grep — run it. A pattern that looks right is not evidence.

## Reasoning about binaries

- Disassemble before asserting instruction length or encoding. Hand-decoding x86 length and ModRM is where confident wrong answers come from.
- An offset is valid for one build. Treat every pasted constant as needing re-verification against the current binary, and prefer PDB resolution or pattern scanning over a hardcoded number.
- Absence of a string is not absence of a feature. Encrypted name pools, stripped metadata, and compressed sections all make greps return nothing while the code is right there.
- A tool that exits 0 did not necessarily do anything. Read its log.

## Extending the pack

New skill: create `skills/<kebab-name>/SKILL.md` with the same frontmatter shape, a self-contained trigger in the first 57 characters of `description`, and a pitfalls section. Long reference material goes in `skills/<name>/references/`, never inline.

Keep the pack's scope: runtime mechanics, tooling, and analysis method. No packed binaries, no per-product bypasses.
