---
name: il2cpp-reverse-engineering
description: Use when reversing Unity IL2CPP games. global-metadata, libil2cpp, Cpp2IL, Il2CppDumper, frida-il2cpp-bridge.
---

# Unity IL2CPP Reverse Engineering

## Toolchain
Cpp2IL, Il2CppDumper, frida-il2cpp-bridge, IDA Pro / Ghidra, dnSpyEx + ILSpy (dummy DLLs), Zygisk/Frida for on-device work.

## How the runtime works

IL2CPP compiles C# to C++ ahead of time. Two artifacts matter:

| Artifact | Contains |
| --- | --- |
| `global-metadata.dat` | type/field/method metadata: names, signatures, offsets |
| `libil2cpp.so` / `GameAssembly.dll` | the compiled native code and runtime |

Metadata file header: magic `0xFAB11BAF`, then a version int (24–31 across Unity 2017–2023+), then file offsets to the string literal pool and the type/field/method definition tables. A mismatch between the metadata version and the tool's supported range is the most common failure — pick a tool release that matches the Unity version.

Version map (approximate): 21 → Unity 5.4, 22 → 5.5, 23 → 5.6, 24 → 2017–2018, 27 → 2019, 29 → 2020, 31 → 2021–2023+.

## Static pipeline

1. **Cpp2IL** is the current standard. `Cpp2IL --game-path <dir> --exe-name GameAssembly.dll --output-as dummydll` (or `--output-as il` for full reconstructed assemblies).
   It handles both the metadata and the compiled binary so offsets come out resolved, which Il2CppDumper no longer does on newer Unity.
2. **Il2CppDumper** still works on older builds: `Il2CppDumper.exe libil2cpp.so global-metadata.dat output/` produces `dump.cs`, `il2cpp.h`, `script.py` (IDA), `ghidra.py`.
3. Import `script.py` into IDA (or `ghidra.py` via `analyzeHeadless -postScript`), apply `il2cpp.h` as types, then rename functions from `dump.cs`.

## Key structures

```c
typedef struct Il2CppClass {
    const Il2CppImage   *image;
    void                *gc_desc;
    const char          *name;
    const char          *namespaze;
    Il2CppType          *byval_arg;
    Il2CppType          *this_arg;
    Il2CppClass          *element_class;
    Il2CppClass          *castClass;
    Il2CppClass          *declaringType;
    const MethodInfo    *declaringMethod;
    Il2CppClass          *parent;
    /* fields follow at a version-dependent offset */
} Il2CppClass;
```

```c
typedef struct Il2CppObject {
    Il2CppClass *klass;
    void        *monitor;     /* the sync block, NOT a C++ vtable */
} Il2CppObject;
```

| Symbol | Role |
| --- | --- |
| `s_GlobalMetadata` | pointer to the mapped metadata blob |
| `s_CodeRegistration` / `s_MetadataRegistration` | the registration structs linking metadata to code |
| `il2cpp_init` | runtime bootstrap; a reliable anchor for static analysis |
| `MethodInfo` | `{ methodPointer, invoker_method, name, klass, return_type, parameters_count, parameters, ... }` |
| `FieldInfo` | `{ type, name, parent (klass), offset, ... }` — **`offset` is the live field offset** |

Because `Il2CppObject` starts with `klass`, finding the class of any instance at runtime is a single dereference. Field access is `*(T*)((char*)obj + FieldInfo->offset)`.

## Runtime extraction

When metadata is embedded, encrypted, or absent from disk, dump it from the live process:

```javascript
// frida-il2cpp-bridge
import "frida-il2cpp-bridge";

Il2Cpp.perform(() => {
  const asm  = Il2Cpp.Domain.assembly("Assembly-CSharp");
  const type = asm.image.class("Player").find();
  console.log(type.fields.map(f => `${f.name} @ +0x${f.offset.toString(16)}`));
  type.methods.forEach(m => {
    Interceptor.attach(m.implementation!, { onEnter(a) { console.log(m.name, a[2]); } });
  });
});
```

`Il2Cpp.perform()` blocks until the domain is initialised — hooking earlier (e.g. before `il2cpp_init` returns) attaches to nothing.

Raw on-device route when the metadata is only in memory: scan `/proc/<pid>/mem` for the `0xFAB11BAF` magic, read the version and offsets from the header, and write the range out. On the desktop the equivalent is a minidump plus a byte scan of the module's `.data`.

## Reconstruction

- `--output-as dummydll` produces stub assemblies. Drop them into dnSpyEx/ILSpy to browse types, fields, and signatures with the real offsets preserved as metadata.
- Full `--output-as il` recovers method bodies where the compiler kept them; expect gaps on newer Unity where code generation differs.
- To find a specific function across builds: match on the `MethodInfo` name/namespace pair from metadata, then resolve its RVA through `s_CodeRegistration->methodPointers`.

## Pitfalls

- **`Il2CppObject` has no vtable.** There is no virtual dispatch through the instance. All dispatch goes through `MethodInfo::methodPointer`, which is why hooking means patching the method pointer or the code at it.
- **`Il2CppClass` layout changes between Unity versions.** Field offsets into the class struct itself drift; only `FieldInfo->offset` (the field's offset inside the *instance*) is stable. Read the class layout from the version you are on.
- **Metadata encryption** — some builds XOR the whole blob or strip the magic. If Cpp2IL reports an invalid magic, the metadata is encrypted; you need a runtime capture.
- **Stripped names** — builds shipped with `--emit-null-checks` but stripped metadata still expose `MethodInfo->name` at runtime. Static-only analysis of a metadata-less build is guesswork.
- **Dummy DLLs do not execute.** Method bodies in `dummydll` output are stubs; they are for browsing, not for running.
- A `global-metadata.dat` that lives *inside* the APK/OBB but is never read from disk (the app mmaps it from the package) means the file on disk may be a decoy or absent entirely — confirm by checking what the runtime actually maps.
