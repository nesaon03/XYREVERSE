---
name: ida-pro-advanced-re
description: Use when doing static binary analysis in IDA Pro. Headless automation, IDAPython, Microcode deobfuscation, type recovery.
---

# IDA Pro Advanced RE

## Toolchain
IDA Pro 7.x / 8.x / 9.x (`idat64`, `ida64`), IDAPython (Python 3.12 on IDA 9), Hex-Rays Decompiler, Lumina, FLIRT, Class Informer.

## Headless / batch

```bash
idat64 -A -S"script.py" -L"log.txt" target.dll   # -A: autonomous, no dialogs
idat64 -B target.dll                              # batch: write .idb/.i64 + .asm
ida64  -A -S"script.py arg1 arg2" target.exe
```

Headless scripts **must** end with `qexit(0)` or IDA idles forever waiting for input. Script arguments arrive via `idc.ARGV` (ARGV[0] is the script path itself).

```python
import ida_auto, idautils, idc, ida_funcs, ida_name, ida_bytes, ida_hexrays

ida_auto.auto_wait()                      # let autoanalysis FINISH before reading the DB
for ea in idautils.Functions():
    f = ida_funcs.get_func(ea)
    print(hex(ea), ida_name.get_name(ea), f.end_ea - f.start_ea)
qexit(0)
```

## Loader and symbol recovery

| Source | How |
| --- | --- |
| PDB | drop beside the binary and load, or File > Load file > PDB |
| MS symbol server | `_NT_SYMBOL_PATH=srv*C:\sym*https://msdl.microsoft.com/download/symbols` then the `pdb` loader |
| MSVC `.map` | File > Load file > MAP |
| DWARF (ELF/Mach-O) | applied on load; re-apply with the `dwarf` plugin |
| FLIRT | `ida_flirt.apply_sig`, or the `flirt` loader |
| Lumina | `ida_lumina` — pulls recovered names/types from Hex-Rays |

## Microcode — deobfuscation at the IR level

`ida_hexrays.mba_t` is a control-flow graph of `mblock_t`, each holding a linked list of `minsn_t`. This is the layer to fold obfuscation in, before the pseudocode is generated.

```python
import ida_hexrays
cf = ida_hexrays.decompile(ea)      # cfunc_t
mba = cf.mba                        # mba_t
for blk in mba:                     # mblock_t
    for ins in blk:                 # minsn_t
        print(ins.opname, ins.dstr())
```

Typical uses: fold MBA identities (`x ^ y + 2*(x & y)` => `x + y`), constant-fold opaque predicates, splice out junk blocks, then `mba.decompile()` and install the cleaned pseudocode.

## Type recovery

```python
import ida_typeinf, ida_struct, idati
ida_typeinf.parse_decls(idati, "struct Foo { int a; char b[8]; };", 0)
tif = ida_typeinf.tinfo_t()
ida_typeinf.parse_decl(tif, idati, "Foo *", 0)
ida_typeinf.apply_tinfo(ea, tif, ida_typeinf.TINFO_DEFINITE)
```

- Local types: Shift-F1. Import `.til` or raw `.h`.
- Decompiler locals are `ida_hexrays.lvar_t`; force one with `lvar.set_lvar_type()` when the decompiler guessed wrong.
- Rebuild vtables with the `Class Informer` plugin (MSVC RTTI) or by walking constructor stores.

## Frequently used IDAPython

```python
idautils.XrefsTo(ea, ida_xref.XREF_ALL)
idautils.CodeRefsFrom(ea, 0)
ida_bytes.bin_search(start, end,
                     "48 8B 05 ? ? ? ?", "48 8B 05 ?? ?? ?? ??",
                     0, ida_bytes.BIN_SEARCH_FORWARD)
ida_name.set_name(ea, "g_MyGlobal", ida_name.SN_NOCHECK)
ida_funcs.reanalyze_function(f)
ida_hexrays.decompile(ea).get_pseudocode()
```

## Pitfalls

- **Autoanalysis is asynchronous.** A script that reads the DB without `auto_wait()` sees a half-built IDB. This is the single most common headless bug.
- `-A` suppresses the "database exists" prompt — point it at a fresh path or copy the `.i64` aside.
- FLIRT and Class-Informer names are *candidates*: identical library code produces false positives. Verify against call sites.
- Tail calls and thunks break the decompiler's local typing; cross-check anything you rely on against the disassembly.
- IDA 9 moved to Python 3.12 and split `idapython.cfg`. Scripts written against IDA 6.x IDC semantics break silently.
- Large binaries (100 MB+) can exhaust memory during autoanalysis — raise `MaxAnalysisTime`, or analyze sections separately.
