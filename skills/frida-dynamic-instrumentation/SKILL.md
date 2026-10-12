---
name: frida-dynamic-instrumentation
description: Use when instrumenting live processes with Frida. Interceptor, Stalker, memory scan/patch, IL2CPP bridging.
---

# Frida Dynamic Instrumentation

## Toolchain
Frida 16+/17+, GumJS, `frida`, `frida-trace`, `frida-ps`, frida-il2cpp-bridge, `frida-gadget`, Zygisk, jadx/apktool for the host app.

## Setup

```bash
frida-ps -Uai                       # device: USB, apps, with identifiers
frida -U -f com.example.app -l hook.js --no-pause     # spawn + inject
frida -U -n GameName -l hook.js                       # attach to a running process
frida-trace -U -i "SSL_*" -f target                   # auto-generated stubs on every match
```

Server-side (rooted device / emulator):

```bash
adb push frida-server-<ver>-android-arm64 /data/local/tmp/frida-server
adb shell "su -c 'chmod 755 /data/local/tmp/frida-server'"
adb shell "su -c '/data/local/tmp/frida-server &'"
```

## Core API

```javascript
const mod = Process.getModuleByName("libgame.so");
const addr = mod.base.add(0x1A2B3C);          // resolve once; offsets are per-build

Interceptor.attach(addr, {
  onEnter(args) {
    this.buf = args[1];                        // save for onLeave
    console.log("arg0 =", args[0].toInt32(), "arg1 =", ptr(args[1]));
    console.log(Thread.backtrace(this.context, Backtracer.ACCURATE)
                      .map(DebugSymbol.fromAddress).join("\n"));
  },
  onLeave(retval) {
    console.log("ret =", retval.toInt32());
    retval.replace(ptr(1));                    // override the return value
  }
});
```

- `args` on ARM64: `args[0..7]` map to `x0..x7`; on x64 `rcx, rdx, r8, r9` then the stack.
- `Interceptor.replace(target, new NativeCallback(...))` swaps the function outright (visible to anyone comparing bytes).
- `Interceptor.attach` writes a trampoline; `Interceptor.detachAll()` restores.
- `Interceptor.flush()` forces pending writes under heavy churn — without it, call sites can be patched lazily and read back stale.
- `Stalker.follow(threadId, { events: { call: true, ret: true } })` traces execution; expensive, use for short windows.
- `Memory.scan(base, size, "48 8B 05 ?? ?? ?? ??", { onMatch, onComplete })`, `Memory.protect(addr, size, 'rwx')`, `Memory.patchCode(addr, size, code => ...)`.
- `NativeFunction(addr, 'int', ['pointer'])` to call into a resolved function, then `.call(...)`.
- `Module.ensureInitialized("libc.so")` (Frida 17+) when you need a module before it is mapped.

## Scripting surface

- **GumJS** (JavaScript) — the default.
- **CModule** — compile C inside the script and inject it; use for loops too hot for JS.

```javascript
const cm = new CModule(`
  #include <gum/gumdefs.h>
  void hello(void) { }
`);
```

- `frida-il2cpp-bridge` (TypeScript, needs a bundler) — Unity IL2CPP specific: `Il2Cpp.perform`, `Il2Cpp.Domain.assembly()`, `type.methods[].implementation` for hooking, `field.offset` for layout.
- Agents can be written in Python/C via `frida-python` + `frida-compile` for anything with real logic.

## Injection without root

| Route | Notes |
| --- | --- |
| `frida-gadget` as a `.so` | repackage the APK, load it from the app's own code path |
| `LD_PRELOAD` | only for your own process tree; breaks under some loaders |
| Zygisk / LSPosed module | system-wide, needs Magisk |
| Patch the app to `dlopen` the gadget | the standard no-root Android route |

Gadget config (`libgadget.config.so` next to the gadget) picks the mode: `listen` (default, connects over the network), `connect` (dials out to a host), `script` (self-contained).

## Android specifics

- `Java.perform(() => { const C = Java.use("com.example.Class"); C.method.implementation = function () { ... } })`.
- `Java.choose("com.example.Class", { onMatch(i) {...}, onComplete() {} })` walks live instances.
- `Process.getModuleByName("libtarget.so")` and AOB scan for offsets that shift per build.
- Hooking `ptrace`, `sysctl`, `/proc/self/status` covers the common user-space anti-debug checks; kernel-level checks (as a separate driver checking `/proc/<pid>/maps` for `frida`) need the process name and thread names disguised, not just the library hidden.

## Pitfalls

- **Offsets are per-build.** Re-derive them; a stale offset fails silently (you hooked nothing) or crashes.
- **Hooking ARM64 code that is executed by a translator** (an AArch64 interpreter running inside an x86 process, or vice versa) does nothing — the host-side hook and the guest-side code are different address spaces.
- Attaching too early: `frida -f` injects before the runtime is up, so IL2CPP/JVM objects do not exist yet. Gate on a ready signal (`Il2Cpp.perform`, or hook the runtime init and continue).
- Tracing every call in a hot loop tanks the game into a stutter; keep `Stalker` scopes short.
- A thread suspended by the debugger cannot be resumed by the hook; `Process.setExceptionHandler` for anything that faults.
- Some loaders strip `LD_PRELOAD` and detect the gadget's maps by name — rename the gadget and patch its strings, and expect a detection pass on `/proc/self/maps`.
