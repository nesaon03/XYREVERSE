---
name: ghidra-headless-analysis
description: Use when automating Ghidra analysis. Headless invocation, PyGhidra, decompiler API, PCode emulation, BSim.
---

# Ghidra Headless Analysis

## Toolchain
Ghidra 11.x, `analyzeHeadless` / `analyzeHeadless.bat`, PyGhidra (Ghidra 11.2+), GhidraBridge, JDK 21.

## Headless invocation

```bash
analyzeHeadless /proj MyProject -import ./target.dll \
  -postScript MyScript.py -scriptPath ./scripts -deleteProject

# Windows
analyzeHeadless.bat C:\proj MyProject -import C:\target.dll ^
  -postScript MyScript.py -scriptPath C:\scripts
```

Useful flags: `-noanalysis`, `-recursive <dir>`, `-overwrite`, `-readOnly`, `-keep`, `-process <name>`, `-preScript`, `-postScript`, `-scriptPath`, `-properties <file>`.

Scripts must resolve on `-scriptPath` or live under `$GHIDRA_HOME/Ghidra/Features/*/ghidra_scripts`.

## PyGhidra vs Jython

- Ghidra <= 11.1 ran `.py` scripts as **Jython 2.7** (Python 2).
- Ghidra 11.2+ ships **PyGhidra**: `.py` runs under real CPython 3 with the `ghidra.*` Java API exposed. Standalone entry point is `pyghidra`; `analyzeHeadless` also honours it.
- Write new scripts for PyGhidra. Jython lacks every modern library and is on the way out.

```python
# @category Analysis
fm = currentProgram.getFunctionManager()
for f in fm.getFunctions(True):
    print(f.getEntryPoint(), f.getName(), f.getBody().getNumAddresses())
```

## Decompiler API

```python
from ghidra.app.decompiler import DecompInterface, DecompileOptions

d = DecompInterface()
d.setOptions(DecompileOptions())
d.openProgram(currentProgram)
res = d.decompileFunction(f, 60, None)          # 60 s timeout, None = no monitor
if res.decompileCompleted():
    print(res.getDecompiledFunction().getC())
```

`res.getHighFunction()` returns the HighFunction: a data-flow graph of `HighVariable` and `PcodeOpAST`. Walk it for def/use chains instead of regexing the C output.

## PCode emulation

The practical way to run a string decryptor or resolver without reimplementing it:

```python
from ghidra.pcode.emu import PcodeEmulator
emu = PcodeEmulator(currentProgram)
emu.writeMemory(arg_addr, data)
emu.getThread(0).stepInstruction()      # or stepOver() / run()
plaintext = emu.readMemory(out_addr, n)
```

`EmulatorHelper` wraps the same engine with register/stack helpers. Pair with a `PcodeInjectLibrary` when the binary uses custom calling conventions.

## Signatures and similarity

- **FunctionID (FID)** — Ghidra's FLIRT analogue. Build a DB with `FunctionIDDatabaseGenerator`, apply through the FID analyzer or a post-script.
- **BSim** — binary similarity across a corpus (bsim server + plugin). Finds the same routine across game builds when names are stripped.
- **Version Tracking** — correlates two program versions and propagates names/types between them.

## Extending

- Java: subclass `GhidraScript`, compile inside the IDE.
- Out-of-process: `ghidra_bridge` / `pyhidra` expose the full API to a normal Python session (notebooks, pipelines).
- Types: `.gdt` archives bundle header sets; build structures programmatically with `StructureDataType` and `DataUtilities`.

## Pitfalls

- Headless fails before parsing arguments if `JAVA_HOME` is unset or points at an unsupported JDK (21 for 11.x).
- A Ghidra project is **locked** by the first writer: a second headless run on the same project blocks. Use `-readOnly` for parallel readers or one project per worker.
- Autoanalysis of a 50 MB+ binary can blow past default timeouts; raise them via `-properties` or split the job by section.
- Jython-era scripts under PyGhidra exit 0 while doing nothing if they contain Python 2 syntax. Read the console log, never the exit code alone.
