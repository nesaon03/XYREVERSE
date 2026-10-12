---
name: dotnet-runtime-re
description: Use when reversing .NET assemblies. dnSpyEx, ILSpy, de4dot, CIL patching, HarmonyX.
---

# .NET Runtime Reverse Engineering

## Toolchain
dnSpyEx, ILSpy (`ilspycmd`), de4dot-cex, AsmResolver, Mono.Cecil, HarmonyX, `ildasm`/`ilasm`, dotnet 8/9.

## Metadata layout

A .NET assembly is a PE file whose CLI header points at metadata tables. Key streams inside the `#~` (or unoptimized `#-`) stream:

| Table | Holds |
| --- | --- |
| `Module` / `Assembly` / `AssemblyRef` | identity of self and referenced assemblies |
| `TypeDef` / `TypeRef` | types and where they come from |
| `MethodDef` / `MethodRef` (`MemberRef`) | methods and call targets |
| `FieldDef` / `FieldRef` | fields |
| `Param` | method parameters plus their names |
| `CustomAttribute` | attributes, resolved through the blob heap |
| `NestedClass` | nesting relationships |

Heaps: `#Strings` (names), `#US` (user strings — every literal in the IL), `#Blob` (signatures and attribute blobs), `#GUID`.

Read them with `System.Reflection.Metadata` (`PEReader` + `MetadataReader`) instead of hand-parsing; it handles malformed input defensively.

```csharp
using var fs = File.OpenRead(path);
using var pe = new PEReader(fs);
var md = pe.GetMetadataReader();
foreach (var h in md.TypeDefinitions) {
    var t = md.GetTypeDefinition(h);
    Console.WriteLine(md.GetString(t.Namespace) + "." + md.GetString(t.Name));
}
```

## CIL patching

Instruction encoding is variable-length: 1-byte opcodes (`nop 0x00`, `ret 0x2A`, `ldarg.0 0x02`), 2-byte opcodes prefixed `0xFE` (`callvirt 0xFE 0x6F`, `brtrue 0xFE 0x2D`), and a 4-byte inline operand for branches, tokens, and constants.

Recipe for a boolean gate — force `Methods.Get("IsLicensed")` to return true:

```
ldc.i4.1    (0x17)
ret         (0x2A)
```

Two edits matter after any body rewrite:

1. **MaxStack** — recompute it, or the verifier rejects the method.
2. **Exception handlers** — offsets in the `EHClause` table shift when instruction lengths change; re-emit the whole body rather than patching lengths in place.

Do this with Cecil/AsmResolver rather than a hex editor:

```csharp
var asm = AssemblyDefinition.ReadAssembly(path);
var m = asm.MainModule.Types.First(t => t.Name == "Gate").Methods.First(x => x.Name == "IsLicensed");
m.Body = new MethodBody(m) {
    Instructions = { Instruction.Create(OpCodes.Ldc_I4_1), Instruction.Create(OpCodes.Ret) },
    MaxStack = 1
};
asm.Write(path + ".patched");
```

## Obfuscation

- **de4dot-cex** strips ConfuserEx, Eazfuscator, SmartAssembly, .NET Reactor, Babel and ~40 more. `de4dot -f file.dll`.
- **String encryption** is usually a static method returning the decrypted literal; call it in a loop over the `#US` heap, or let de4dot's emulator trace and inline it.
- **Control-flow flattening** shows up as a `while(true) switch(state)` dispatcher with a state variable. Symbolically execute or emulation-trace the dispatcher; `ControlFlowDeobfuscation` in de4dot handles the common shapes.
- **Name mangling** is cosmetic: strip non-printable characters when searching, and keep a rename map as you go.
- **Anti-tamper / virtualization** (Themida-.NET, VMProtect-.NET, Eazfuscator native part) pushes methods into a native stub. That is a native-RE problem — switch to IDA on the stub and treat the managed side as a caller.

## Runtime

- `HarmonyX` detours at runtime: `harmony.Patch(original, prefix, postfix, transpiler)`. A `Transpiler` rewrites the IL instruction list of the target before JIT — the cleanest way to modify behavior without touching the file on disk.
- `Mono.Cecil` + a profiling/startup hook (`COR_PROFILER`, `AssemblyResolve`) to load a patched assembly in memory.
- Debugging: dnSpy's debugger edits IL live and lets you step the modified result; `dotnet-dump` + SOS (`!clrstack`, `!dumpheap -stat`) for post-mortem analysis.

## Pitfalls

- **Strong names** — editing a signed assembly breaks the signature. Either re-sign with a new key (calls to the original from elsewhere break) or strip the strong-name check by patching `Assembly.GetPublicKey`
- **NGen / ReadyToRun / AOT** — R2R images carry precompiled native code. Patching the IL alone does nothing until the runtime falls back; either `DOTNET_ReadyToRun=0` or patch the native body.
- **Mixed assemblies** (C++/CLI) have both an IL body and native code; ILSpy shows only the IL half.
- Typos in metadata row indices shift every subsequent row — always validate by reloading the assembly after a write.
- ILSpy's "decompile to project" plus `dotnet build` is often faster than byte-patching when the assembly is unobfuscated.
