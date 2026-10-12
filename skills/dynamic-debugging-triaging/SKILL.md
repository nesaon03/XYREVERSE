---
name: dynamic-debugging-triaging
description: Use when debugging live binaries in x64dbg. Anti-debug bypass, ScyllaHide, crash triage, unpacking.
---

# Dynamic Debugging and Crash Triaging

## Toolchain
x64dbg / x32dbg, ScyllaHide (+ TitanHide kernel companion), Scylla (IAT rebuild), Detect It Easy (DIE), WinDbg (kernel), PE-bear.

## Session setup

```
Options > Preferences   -> "Don't show this dialog again", event log verbosity
Options > Events        -> uncheck entry breakpoint spam, keep DLL load notices for the target only
Plugins > ScyllaHide    -> options for the profile (see below)
```

Trace: `Trace > Trace into` (`Ctrl+Alt+F7`) with `Log text`/`Log file` set — the log is a call log you can grep, far faster than stepping.

Hardware breakpoints (`Ctrl+Alt+B`) write debug registers, never memory, so they survive both `VirtualProtect` checks and self-CRC. Software breakpoints (`F2`) write `0xCC` into the code and are both detectable and invisible to a CRC computed after the write.

## Anti-debug

| Check | Where | Bypass |
| --- | --- | --- |
| `IsDebuggerPresent` | `kernel32` → PEB `BeingDebugged` | patch the return, or clear `PEB+2` |
| `CheckRemoteDebuggerPresent` | `NtQueryInformationProcess(ProcessDebugPort)` | return 0 |
| `NtGlobalFlag` | `PEB+0x68` (x64) / `+0x68` flags `0x70` | clear the flags |
| `ProcessDebugFlags` | `NtQueryInformationProcess(0x1F)` | return 1 |
| `NtSetInformationThread(ThreadHideFromDebugger)` | thread hides itself | stub the call |
| `NtQueryObject(ObjectTypeInformation)` | counts open handles to `DebugObject` | return an empty buffer |
| heap flags | `PEB->ProcessHeap->Flags` | clear `HEAP_TAIL_CHECKING_ENABLED` etc. |
| `rdtsc` timing | measures single-step latency | raise the compare threshold |
| SEH/VEH tricks | `INT3` + handler, or `INT 0x2D` | let it throw; the handler eats it |
| self-CRC | hashes `.text` at runtime | use hardware BPs, or patch the compare after the hash |
| guard pages | `PAGE_GUARD` on a code path | single-step past with `PAGE_GUARD` handling on |
| TLS callbacks | run before the entry point | break on `LdrpCallTlsInitializers` |

ScyllaHide profiles: `VMProtect`, `Themida`, `Obsidium`, `SafeEngine`, or a hand-built set of the hooks above. `TitanHide` moves the same hooks into the kernel, needed when the check is a driver calling `NtQueryInformationProcess` on itself.

## Unpacking

1. **Find the OEP.** Options: "Trace into" from the packer entry and watch for the section jump; ESP-trick (set a hardware BP on the stack slot holding the return address before the last `pushad`); or scan the last section for the JMP that lands in the original `.text` range.
2. **Dump.** `Scylla` → select process → `IAT Autosearch` → `Get Imports` → `Dump`. For partial dumps use `Plugins > x64dbg Scylla` while the process is still at the OEP, before the packer restores anything.
3. **Rebuild the IAT.** Fix unresolved thunks by hand, then `Fix Dump`.
4. Nested packers (VMProtect inside a molebox wrapper) need the first layer dumped, then the dumped file unpacked again as a fresh target.

Some packers write `IsDebuggerPresent` results into a checksum (VMProtect scatters anti-debug into the VM handlers). Prefer hooking the syscall layer over patching individual call sites.

## Crash triage

1. **Read the exception code first.**
   - `0xC0000005` access violation — note read vs write and the address: `0x0` = null deref, a small offset from 0 = bad struct field, a wild pointer = use-after-free / bad vtable.
   - `0xC000001D` illegal instruction — wrong architecture or corrupted code.
   - `0xC0000094` integer divide by zero; `0xC00000FD` stack overflow.
2. **Call stack** — `Alt+K` (x64dbg call stack) or `k` in WinDbg. A stack with one frame after a hook means the hook corrupted the stack (wrong calling convention, or a `ret` where `ret 8` was needed).
3. **`.pdata`/`.eh_frame`** — a missing unwind entry makes the walker stop early, so a short stack is not proof the frames do not exist.
4. **Minidump** — `procdump -ma -e -w <name>` or `MiniDumpWriteDump`. Load it in WinDbg: `!analyze -v`, `!clrstack`/`!dumpheap` for .NET, `.ecxr` to move to the faulting context.
5. **Compare against a working run** — the same trace on a clean build isolates what your patch changed.

## Reading a hang

- Deadlock: threads in `WaitForSingleObject`/`NtWaitForSingleObject` with a lock held by a thread that is itself blocked.
- Infinite loop: attach, pause, read `RIP` twice a second apart — the same or a nearby address means a spin.
- I/O wait: check the handle table (`!handle`) for what the thread is blocked on.

## Pitfalls

- Software breakpoints change the bytes and get caught by any self-check. Use hardware BPs for anything that hashes itself.
- `VirtualProtect` + `FlushInstructionCache` dance: patching without the flush means the CPU can execute a stale line.
- A debugger attached at load changes the process's timing and heap layout — bugs that only appear in a debugger are usually uninitialized-memory reads.
- x64dbg's "ignore first-chance exceptions" hides the real fault under a wrapper that catches everything; turn it off when triaging.
- Scylla's IAT autosearch produces plausible garbage on a fully virtualized binary — validate each thunk against the resolved module range before fixing the dump.
