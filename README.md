# ReClass.NET-PS5DebugPlugin

A ReClass.NET plugin that talks to [ps5debug-NG](https://github.com/Pharaoh2k/ps5debug-NG),
a debugger payload for jailbroken PlayStation 5 consoles. Ported from
[TetzkatLipHoka/ReClass.Net-PS4DebugPlugin](https://github.com/TetzkatLipHoka/ReClass.Net-PS4DebugPlugin).

This requires:
- A jailbroken PS5 with `ps5debug-NG.elf` loaded (see the ps5debug-NG README for deployment).
- ReClass.NET (https://github.com/KN4CK3R/ReClass.NET) to load this plugin into.

## Why this port was mostly mechanical

ps5debug-NG's core process/debug/kernel/console commands are wire-compatible
with the original ps4debug protocol this plugin was written against: same
opcodes (`0xBDAA0001..`, `0xBDBB0001..`, `0xBDCC0001..`, `0xBDDD0001..`), same
packet/struct sizes (`proc_list_entry` 36B, `proc_vm_map_entry` 58B,
`cmd_proc_info_response` 188B, GP regs 176B, FPU/YMM 832B, dbregs 128B,
thread-info 40B), and the same on-the-wire status word values
(`CMD_SUCCESS=0x80000000`, `CMD_ERROR=0xF0000001`, etc.). Ports 744 (command),
755 (async debug interrupts) and the UDP discovery beacon (1010, magic
`0xFFFFAAAA`) are unchanged too. See `PROTOCOL.md` in the ps5debug-NG repo for
the authoritative reference this port was checked against.

## What changed from the PS4 plugin

- Renamed `PS4DBG` -> `PS5DBG`, `PS4DebugPlugin` -> `PS5DebugPlugin`,
  settings file `PS4.ini` -> `PS5.ini`, and all related UI labels/identifiers.
- `MAX_BREAKPOINTS` raised from 10 to 30 (ps5debug-NG's `CMD_DEBUG_SET_BREAKPOINT`
  accepts index 0-29; the PS4 server only had 10 slots).
- Added `GetConsoleFirmwareVersion()` (`CMD_FW_VERSION`, e.g. `900` = FW 9.00)
  and `GetConsoleBranding()` (`CMD_BRANDING`) - two ps5debug-NG-only info
  commands with no PS4Debug equivalent. Neither is wired into the UI; call
  them yourself if you want to surface firmware/branding info in a settings
  panel.
- Everything else (memory read/write, process list/maps/info, alloc/free,
  protect, legacy value scan, debugger attach/detach, software breakpoints,
  hardware watchpoints, register get/set, kernel read/write, console
  reboot/print/notify) is a straight rename with no protocol changes, because
  the wire format didn't change.

## Not ported (newer ps5debug-NG-only features)

ps5debug-NG adds a large amount of functionality with no ps4debug counterpart
that this plugin does **not** use: the iterative/Turbo scan families, AOB
scans, `CMD_PROC_CALL`/ELF injection helpers, the built-in Zydis disassembler
and Keystone assembler, server-side stack walk, FS/GS base get/set, and the
kernel-log forwarder on port 3232. Adding wrappers for these is straightforward
(same `SendCMDPacket`/`ReceiveData` pattern as the existing methods) but is
out of scope for a like-for-like port - see `PROTOCOL.md` in the ps5debug-NG
repo (`common/include/protocol.h` for the exact struct layouts) if you want to
extend this further.

## Licensing note

ps5debug-NG is licensed GPL-3.0. This client-side plugin implements the same
public wire protocol documented in that project's `PROTOCOL.md` (opcodes,
struct layouts) rather than incorporating any of its C source, in the same
way the original PS4Debug plugin implemented the ps4debug wire protocol
without embedding ps4debug's own source. The plugin itself carries the same
license as the upstream ReClass.NET-PS4DebugPlugin project (MIT, see LICENSE).
If you redistribute a build that also bundles or links ps5debug-NG code
itself (as opposed to just speaking its wire protocol), that portion remains
subject to GPL-3.0.

## Building from GitHub (no local Visual Studio needed)

This zip includes `.github/workflows/build.yml`. If you push this folder to
your own GitHub repo (named `ReClass.NET-PS5DebugPlugin` to match the solution
name, though any name works), the workflow will:

1. Check out [ReClassNET/ReClass.NET](https://github.com/ReClassNET/ReClass.NET)
   and this repo as sibling folders (`ReClass.NET\` and
   `ReClass.NET-PS5DebugPlugin\`), matching the relative project reference in
   `PS5DebugPlugin.sln`.
2. Restore NuGet packages and run MSBuild on a `windows-latest` runner for
   both `x86` and `x64`.
3. Upload `PS5DebugPlugin.dll` as a workflow artifact you can download from
   the Actions run summary.

To use it:
```
git init
git add .
git commit -m "Initial port to ps5debug-NG"
git remote add origin https://github.com/<you>/ReClass.NET-PS5DebugPlugin.git
git push -u origin main
```
Then check the "Actions" tab on your repo - it runs automatically on push, or
you can trigger it manually via "Run workflow" (the `workflow_dispatch`
trigger). Grab the built DLL from the run's "Artifacts" section and drop it
into ReClass.NET's `x86\Plugins` or `x64\Plugins` folder.

No changes needed to make this work - the workflow is already wired to the
exact folder-naming convention this plugin (and ReClass.NET's other
official plugins) expects.

## Compiling

If you want to compile the ReClass.NET Plugins just fork the repository and
create the following folder structure. If you don't use this structure you
need to fix the project references.

```
..\ReClass.NET\
..\ReClass.NET\ReClass.NET\ReClass.NET.csproj
..\ReClass.NET-PS5DebugPlugin
..\ReClass.NET-PS5DebugPlugin\PS5DebugPlugin.sln
```
