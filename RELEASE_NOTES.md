## XYREVERSE v1.0.0

Nine agent skills for reverse engineering. Each is a self-contained `SKILL.md` an AI agent loads on demand: toolchain, working commands, and a pitfalls list written from failures rather than documentation.

### Skills

| Skill | Triggers on |
| --- | --- |
| `ida-pro-advanced-re` | Static analysis in IDA Pro — headless batch, IDAPython, Microcode, type recovery |
| `ghidra-headless-analysis` | Headless Ghidra — PyGhidra, decompiler API, PCode emulation, BSim |
| `kernel-driver-internals` | Windows kernel drivers — IRP dispatch, IOCTL mechanics, physical memory, WinDbg |
| `dotnet-runtime-re` | .NET assemblies — metadata tables, CIL patching, de4dot, HarmonyX |
| `il2cpp-reverse-engineering` | Unity IL2CPP — global-metadata, libil2cpp, Cpp2IL, frida-il2cpp-bridge |
| `unreal-engine-dumping` | Unreal SDK dumps — GObjects, FNamePool, UWorld traversal, Dumper-7 |
| `frida-dynamic-instrumentation` | Live instrumentation — Interceptor, Stalker, memory scan/patch, gadget |
| `dynamic-debugging-triaging` | x64dbg — anti-debug bypass, ScyllaHide, unpacking, crash triage |
| `rea-automated-analysis` | Driving REA — CLI and MCP, script discovery, app graph extraction |

### Install

```bash
# Hermes
cp -r skills/* ~/AppData/Local/hermes/skills/

# Claude Code / generic agent
cp -r skills/* ~/.claude/skills/
```

Each `description` puts a standalone trigger in its first 57 characters — that is often all a router sees before deciding to load the file.

### Scope

Runtime mechanics, tooling, and analysis method: how a debugger attaches, how a runtime lays out its objects, how a driver receives an IRP. No packed binaries, no game assets, no vendor tooling, no per-product bypasses.

MIT licensed.
