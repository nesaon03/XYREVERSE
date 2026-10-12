---
name: kernel-driver-internals
description: Use when reversing or writing Windows kernel drivers. IRP dispatch, IOCTL mechanics, physical memory primitives, WinDbg.
---

# Windows Kernel Driver Internals

## Toolchain
WDK 10.0.26100, EWDK, WinDbg (x64), LiveKd, IDA/Ghidra with a PDB-loaded `ntoskrnl.exe`. Baselines: Win10 19045, Win11 26100 (24H2).

## Driver skeleton

```c
NTSTATUS DriverEntry(PDRIVER_OBJECT drv, PUNICODE_STRING reg) {
    UNICODE_STRING name = RTL_CONSTANT_STRING(L"\\Device\\MyDev");
    PDEVICE_OBJECT dev;
    NTSTATUS st = IoCreateDevice(drv, 0, &name, FILE_DEVICE_UNKNOWN, 0, FALSE, &dev);
    if (!NT_SUCCESS(st)) return st;

    drv->MajorFunction[IRP_MJ_CREATE]         = DispatchCreate;
    drv->MajorFunction[IRP_MJ_CLOSE]          = DispatchCreate;
    drv->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DispatchIoctl;
    drv->DriverUnload = DriverUnload;

    dev->Flags |= DO_BUFFERED_IO;
    return STATUS_SUCCESS;
}
```

Load from user mode: `sc create MySvc type= kernel binPath= C:\path\My.sys` then `sc start MySvc`. `NtLoadDriver` takes the same path but requires `SeLoadDriverPrivilege`.

## IRP dispatch

Two entry points matter: `IRP_MJ_DEVICE_CONTROL` (user `DeviceIoControl`) and `IRP_MJ_INTERNAL_DEVICE_CONTROL` (kernel `IoDeviceControl`). Everything the handler needs is in the stack location:

```c
PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
ULONG code = sp->Parameters.DeviceIoControl.IoControlCode;
ULONG inLen  = sp->Parameters.DeviceIoControl.InputBufferLength;
ULONG outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
PVOID buf    = Irp->AssociatedIrp.SystemBuffer;      // buffered method
```

The buffer method is encoded in the IOCTL itself via `CTL_CODE(Function, Method, Access, Index)`:

| Method | Input | Output |
| --- | --- | --- |
| `METHOD_BUFFERED` | `SystemBuffer` | `SystemBuffer`; set `Irp->IoStatus.Information` to bytes written |
| `METHOD_IN_DIRECT` | `SystemBuffer` | MDL at `Irp->MdlAddress`, use `MmGetSystemAddressForMdlSafe` |
| `METHOD_OUT_DIRECT` | `SystemBuffer` | same MDL path |
| `METHOD_NEITHER` | `Parameters.DeviceIoControl.Type3InputBuffer` | `Irp->UserBuffer` — must `ProbeForRead/ProbeForWrite` inside `try/except` |

`IoCompleteRequest(Irp, IO_NO_INCREMENT)` runs exactly once per IRP, at IRQL <= DISPATCH_LEVEL. Return `STATUS_PENDING` only after `IoMarkIrpPending`.

## Memory primitives

```c
void *p = ExAllocatePool2(POOL_FLAG_NON_PAGED, size, 'gaTD');
```

- `ExAllocatePool2` supersedes `ExAllocatePoolWithTag`; tags are four bytes, reversed by the macro.
- `MmCopyMemory(dst, (PVOID)phys, len, MM_COPY_MEMORY_PHYSICAL, &copied)` — physical reads, works above 4 GB.
- `MmMapIoSpaceEx(phys, len, PAGE_READWRITE)` — maps physical memory into the kernel VA space.
- `MmGetSystemAddressForMdlSafe(mdl, NormalPagePriority)` for MDL-backed buffers.
- `MmProbeAndLockPages` / `MmUnlockPages` to pin a user buffer you hold across a wait.

IRQL is the rule that breaks the most code: file I/O, registry access, and any waiting `Zw*` call need `PASSIVE_LEVEL`; DPCs and spinlock bodies run at `DISPATCH_LEVEL`; anything touched above `APC_LEVEL` must be nonpaged.

## Address translation (4-level x64, 48-bit VA)

```
CR3 -> PML4E[47:39] -> PDPTE[38:30] -> PDE[29:21] -> PTE[20:12]
```

- `PDE.PS` bit 7 set = 2 MB large page; physical base = `PDE & 0x000FFFFFFFE00000` | offset[20:0].
- 4 KB path: PFN in PTE bits 12–51, flags in the low 12 (P / RW / US / NX).
- To address sub-pages inside a 2 MB mapping, allocate a page table, fill 512 PTEs pointing at the original 4 KB frames, rewrite the PDE with the table's PFN and `PS` cleared, then `__invlpg` the range.
- `MiGetPteAddress`, `MiGetPdeAddress` and the self-map are all PDB-resolvable. Resolve from the PDB — never carry hardcoded offsets across builds.

## What the OS records

- `NtLoadDriver` → `IopLoadDriver`; the image is mapped by a section object and `SeValidateImageHeader` (CI.dll) decides on the signature.
- Loaded modules are enumerated by walking `PsLoadedModuleList`. Unloaded drivers keep a name record in `MmUnloadedDrivers`.
- Driver objects live in the object manager under `\Driver\<name>`, reachable from `ObpRootDirectoryObject` or `ObReferenceObjectByName`.
- `PiDDB` is the PnP driver database — a record of driver *files* the PnP manager has seen on disk, not of what is loaded right now. Forensically it is an artifact-of-history table.
- WinDbg: `!drvobj \Driver\MyDev 7`, `!devobj`, `!irp <addr>`, `!object \Device`, `!pool <addr>`, `lm`, `!chkimg`.

## Static analysis

Pull `ntoskrnl.exe` out of `C:\Windows\System32` and let IDA or Ghidra fetch the PDB from `https://msdl.microsoft.com/download/symbols`. Every internal `Mi*` / `Mm*` / `Ob*` / `Io*` / `Ke*` routine then resolves by name, which is the difference between reading a driver and guessing at one.

For offsets that must survive a build change, resolve at runtime through the PDB with the DIA SDK (`NoRegCoCreate("msdia140.dll", ...)` → `loadDataFromPdb` → `findChildren(SymTagFunction)` → `get_relativeVirtualAddress`), or pattern-scan the routine prologue.

## Pitfalls

- `IoCreateDevice` with an existing name returns `STATUS_OBJECT_NAME_EXISTS`. Delete the symbolic link and device on unload or the next load fails.
- An unload path that leaks an allocation keeps the driver image referenced and the service un-stoppable.
- `METHOD_NEITHER` handlers are the classic arbitrary-write source. Always `ProbeForWrite` and wrap the copy in `try/except`.
- Completion routines run at `DISPATCH_LEVEL`; their context must be nonpaged, allocated before the IRP is sent down.
- The compiler reorders plain loads and stores. Use `READ_ONCE` / `WRITE_ONCE` semantics for flags shared with an ISR or DPC.
- WDK headers change `ExAllocatePool*` deprecation status between releases — check the build warning, not a blog post.
