---
name: unreal-engine-dumping
description: Use when dumping Unreal Engine SDKs. GObjects, FNamePool, UWorld traversal, Dumper-7, UE4SS.
---

# Unreal Engine Dumping

## Toolchain
Dumper-7, UE4SS, IDA (Class Informer), Cheat Engine, UE 4.18 → 5.5.

## Object model

Everything derives from `UObject`. The global object array and name pool are the two roots you need.

```cpp
struct FUObjectItem {
    UObject* Object;
    int32_t  Flags;
    int32_t  ClusterRootIndex;
    int32_t  SerialNumber;
};

struct FUObjectArray {          // "GObjects"
    int32_t   ObjFirstGCIndex;
    int32_t   ObjLastNonGCIndex;
    int32_t   MaxObjectsNotConsideredByGC;
    bool      OpenForDisregardForGC;
    TArray<FUObjectItem*> ObjectPtrs;   /* UE4.18+ same layout, different chunking */
};
```

Layout history (this is where most dumps go wrong):

| Version | Shape |
| --- | --- |
| <= UE 4.10 | a single `TArray<FUObjectItem>` |
| 4.11 – 4.17 | fixed-size chunks, `Objects[chunk][index]` |
| **4.18+ through 5.x** | `FUObjectArray` holds a growable set of **chunks**; index → chunk = `index / 0x10000`, item = `chunk[index % 0x10000]` |

Chunk size is `0x10000` items for the main arrays. `TUObjectArray` is the wrapper; `Num()` is at `+0x??` and `Objects` (the chunk pointer array) next to it.

## Name pool

`FName` is a pair of ids (`ComparisonIndex`, and `Number` for the `_1`, `_2` suffix). Names live in an `FNamePool`:

```cpp
struct FNameEntryHeader {          /* packed into one uint16 */
    uint16 IsWide : 1;
    uint16 Len    : 15;
};
struct FNameEntry {
    FNameEntryHeader Header;
    union { char ansi[1]; wchar_t wide[1]; };
};

struct FNamePool {
    /* UE 4.23+ */ FNameEntryAllocator Entries;
    /* UE 5.0+  */ uint32 AnsiCount, WideCount;
};
struct FNameEntryAllocator {
    FRWLock  Lock;                 /* size drifts across builds — do not hardcode */
    uint32   CurrentBlock;
    uint32   CurrentByteCursor;
    uint8*   Blocks[8192];         /* 64 KB blocks */
};
```

Index decode: `block = index >> 16`, `offset = index & 0xFFFF` (times the stride: 2 for ansi-on-4-byte stride in 4.23+, 4 for UE 4.23 with the 2-byte header). Every one of those parameters changed at least once; read them out of the binary rather than assuming.

**XOR-encrypted names** (UE 4.23–5.0 era): the resolved pointer is XORed with a per-build key, and the entry's own bytes are XORed with a rolling value derived from the name index. Dumper-7 carries a `FNamePoolDecrypt` implementation; the key is patchable in memory.

## Find the roots

Static:

1. Load the shipping exe in IDA, let the PDB/FName strings anchor you.
2. `GObjects` is referenced by `UObject::ProcessEvent`, `StaticFindObject`, and the GC; find `StaticFindObject` via its string `"Failed to find object"` and walk up to its global.
3. `GNames`/`FNamePool` is referenced by `FName::FName(wchar_t*)` and `FName::ToString`.
4. `ProcessEvent` itself is a virtual at a fixed vtable index (often `0x44`/`68` in UE4, `0x46`/`70` in UE5) — that index is the reliable, version-stable anchor for calling into the engine.

Runtime: walk the `UWorld` chain.

```
UWorld* World = *(UWorld**)(engineBase + GWorld_offset);   // or via GEngine->GameViewport->World
ULevel* Level = World->PersistentLevel;
TArray<AActor*> Actors = Level->Actors;
```

`UWorld` holds `PersistentLevel`, `Levels`, `OwningGameInstance`, `GameState`; `ULevel` holds `Actors`, `ActorsForGC`; `APlayerController` holds `AcknowledgedPawn`, `PlayerCameraManager`.

## SDK generation

Dumper-7 (current, UE 4.18 → 5.5): inject or attach, it walks objects and emits a full SDK — class hierarchy, struct layouts with padding, enum values, bitfield widths, function parameter names, and vtable indices.

Validation checklist before trusting a generated SDK:

1. Dump a known class and check `sizeof` against the field list plus padding.
2. Compare `Objects[5].InternalIndex == 5` — Dumper-7 asserts this to confirm the object array is parsed correctly.
3. Resolve one known function through its vtable index and call it; a wrong index crashes.
4. Verify a UFunction's `ParmsSize` against the parameter list.

UE4SS (Lua/C++ scripting) covers the runtime side: `RegisterHook`, live object browsing, `UE4SS_Signatures` for per-build AOBs.

## Calling into the engine

```cpp
using ProcessEventFn = void(*)(UObject*, UFunction*, void*);
((ProcessEventFn)(base + ProcessEvent_RVA))(obj, func, &params);
```

Params are a struct whose layout matches the UFunction's parameter order; `func->FunctionFlags & FUNC_Native` decides whether the native thunk runs instead. For `UFUNCTION`s this is the whole cheat surface: find the UFunction by name in the `GObjects` walk, then call it.

## Pitfalls

- **Offsets are per-build.** The engine version tells you the *shape*, not the numbers. Resolve `GObjects`, `GNames`, and `ProcessEvent` per binary — AOB scan or PDB, never a pasted constant.
- `FName` comparison is by **index**, not string. Two identical strings can have different ids if they came from different pools; compare `ComparisonIndex`.
- Reading names during a GC pass crashes; suspend the thread or gate on `FUObjectArray::IsOpenForDisregardForGC`.
- `TArray` is `{ data, Num, Max }` — never read `Num` items without checking `Max`, and never cache `data` across a level load.
- On UE5, `FName` got `DisplayIndex`; code that assumes a 2-field `FName` reads garbage.
- Encrypted name pools make string searches return nothing. Do not conclude "no names" — check whether the pool is XORed first.

## Dumping on the analyst's side

- **Dumper-7** — needs the process running, an SDK dump per build; keep the generated headers next to the offsets module.
- **UE4SS** — has a "dump objects" mode and live UFunction invocation, useful for verifying a UFunction exists before building around it.
- **Static only** — with no PDB, `GObjects` is found via the `"FUObjectArray::AllocateUObjectIndex"` string or by scanning for a pointer array whose elements all point at objects whose first field points back at a shared `UClass` layout. This is the slow path; prefer a runtime dump.
