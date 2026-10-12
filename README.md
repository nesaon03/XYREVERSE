# XYREVERSE

Agent skill pack for reverse engineering. Nine skills an AI coding agent can load on demand — static analysis, dynamic instrumentation, runtime internals, SDK dumping, and automation.

Built for agents that already have shell and file tools. Each skill is a self-contained `SKILL.md` with a one-line trigger in its frontmatter `description`, a toolchain section, working commands, and a pitfalls list written from things that actually broke.

## Skills

| Skill | Triggers on |
| --- | --- |
| [`ida-pro-advanced-re`](skills/ida-pro-advanced-re/SKILL.md) | Static analysis in IDA Pro — headless batch, IDAPython, Microcode, type recovery |
| [`ghidra-headless-analysis`](skills/ghidra-headless-analysis/SKILL.md) | Headless Ghidra — PyGhidra, decompiler API, PCode emulation, BSim |
| [`kernel-driver-internals`](skills/kernel-driver-internals/SKILL.md) | Windows kernel drivers — IRP dispatch, IOCTL mechanics, physical memory, WinDbg |
| [`dotnet-runtime-re`](skills/dotnet-runtime-re/SKILL.md) | .NET assemblies — metadata tables, CIL patching, de4dot, HarmonyX |
| [`il2cpp-reverse-engineering`](skills/il2cpp-reverse-engineering/SKILL.md) | Unity IL2CPP games — global-metadata, libil2cpp, Cpp2IL, frida-il2cpp-bridge |
| [`unreal-engine-dumping`](skills/unreal-engine-dumping/SKILL.md) | Unreal SDK dumps — GObjects, FNamePool, UWorld traversal, Dumper-7, UE4SS |
| [`frida-dynamic-instrumentation`](skills/frida-dynamic-instrumentation/SKILL.md) | Live instrumentation — Interceptor, Stalker, memory scan/patch, gadget |
| [`dynamic-debugging-triaging`](skills/dynamic-debugging-triaging/SKILL.md) | x64dbg sessions — anti-debug bypass, ScyllaHide, unpacking, crash triage |
| [`rea-automated-analysis`](skills/rea-automated-analysis/SKILL.md) | Driving REA — CLI and MCP, script discovery, app graph extraction |

## Install

Copy or symlink `skills/` into your agent's skill directory:

```bash
# Hermes
cp -r skills/* ~/AppData/Local/hermes/skills/

# Claude Code / generic
cp -r skills/* ~/.claude/skills/
```

An agent picks a skill when its `description` matches the task. The first 57 characters of each description are written to work standalone, because that is often all a router sees.

## Layout

```
skills/<name>/SKILL.md      frontmatter (name, description) + body
AGENTS.md                   how an agent should load and use this pack
LICENSE                     MIT
```

Add supporting files under `skills/<name>/references/` for long reference material — keep `SKILL.md` as the trigger and the working procedure.

## Scope

Everything here is mechanics: how a debugger attaches, how a runtime lays out its objects, how a driver receives an IRP. No packed binaries, no game assets, no vendor tooling, no bypasses for specific products. Bring your own target and your own authorization.

## License

MIT — see [LICENSE](LICENSE).
