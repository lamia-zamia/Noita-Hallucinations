# Native crash reports on Windows

The January 25, 2025 developer build, `noita_dev.exe`, writes a minidump under
`crashes/` in the game directory. The filename includes the build date, local
time, process ID and faulting thread ID. Its top-level exception filter
(`0x005b83d0`) passes the original exception pointers to Windows'
`MiniDumpWriteDump`. It requests `MiniDumpWithDataSegs` and returns
`EXCEPTION_CONTINUE_SEARCH`, so Windows can also record the crash. The retail
`noita.exe` from the same release has no corresponding dump-writing import
or filename template. Windows Error Reporting can be a separate dump source.
[Microsoft's documentation on local dumps](https://learn.microsoft.com/en-us/windows/win32/wer/collecting-user-mode-dumps)

There are three useful artifacts:

| Artifact | What it can tell you |
|---|---|
| `crashes/*.dmp` | Exception code, faulting thread and instruction, registers, and native call stack. Use the executable and DLLs from the same build. |
| `log_asserts.txt` in the developer build | Engine assertions with source filenames and line numbers. Match the timestamp to the dump. A recorded assertion may be nonfatal. |
| `logger.txt` | Messages from the game's ordinary logger. Later launches can replace the file, so retain the copy from the failed run. |
| Steam's `logs/gameprocess_log.txt` | The exit code of every run, by process ID, even when no dump was written. |

When there is no dump, the exit code tells how the game ended. `0` is a normal
quit: every `exit` call in the engine passes 0 (ten sites in each build, plus
the C runtime passing `main`'s result). `0xC0000005` is an access violation,
`0xC00000FD` a stack overflow, `0xC0000409` a fail-fast, and `0xC000013A` means
the developer build's console window was closed. Fail-fast exits never reach
exception handlers, so no in-process recorder sees them. Exit code `1` is what
Task Manager's End task leaves, and also what LuaJIT's panic leaves (below);
the engine itself does not pass 1 to `exit`.

LuaJIT's own exit also bypasses exception handlers. After an error outside any
protected call it prints `PANIC: unprotected error in call to Lua API (...)` to
the console and calls `exit(1)` from `lua51.dll`, which imports `exit` from
`MSVCR120.dll`. A recorder can hook that import instead. At all three `exit`
calls in the shipped `lua51.dll` (two panic paths and `os.exit`), the
`lua_State` is still in `EDI` or `ESI`, so the error message and the state's
stack size can be read there. LuaJIT is built without frame pointers, so a
stack walk has to find the engine's frames by scanning.

The Lua traceback the game logs after `Lua error - ` (`0x007edd40` retail) is
cut past 22 levels to the first 11, `...`, and the last 10. An empty
traceback, `Stack traceback:` with no lines, means the failing Lua state had no
function running at level 1.

Open the dump in a Windows debugger. `.exr -1` shows the fault and its
parameters. `.ecxr` restores the faulting thread's registers, then `kb` shows
its native callers. Record addresses as module plus offset. The retail and
developer executables have different offsets, and without private symbols a
debugger may label an unrelated game frame as a large offset from the nearest
export. A small dump can preserve a pointer while omitting the Lua state or
mod record it points to.
[Microsoft's guide to user-mode dumps](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/analyzing-a-user-mode-dump-file)

The developer executable includes native profiling labels in some
function bodies. Both builds also contain C++ RTTI for virtual classes.
Those records can identify a function or its owning class using the
matching executable even when the debugger has only exported names.

## Querying biome-map size before the world exists

One startup failure in the developer build enters `BiomeMapGetSize()`
(`0x00830970`) during the
[`OnMagicNumbersAndWorldSeedInitialized` callback](lua-execution.md#when-each-callback-fires-and-in-what-order).
The API checks for a live Lua biome-map context. With no context it reads
the dimensions from the world's procedural terrain. In the observed crash the
world pointer is null, so the terrain lookup at `0x00661180` faults reading
address `0x48`. The developer assertion names `mGameWorld` in
`source/game_globals.cpp:678`. Retail has the same unguarded fallback at
`0x007c7871`.

Query the size during biome-map generation when its Lua context is active, or
after the world is initialized. A Lua `pcall` handles a Lua error. The null
read above raises a native access violation even though the engine already
invoked the callback through `lua_pcall`.

Evaluating a biome-map script manually, even in a separate Lua environment,
leaves the engine's biome-map context inactive. An appended chunk can reach
the native `BiomeMapGetSize()` unless the replay supplies its own
implementation. The Lua state running the chunk can belong to a different
mod from the one that supplied the append.

## Missing world state during Verlet physics

In the January 25 developer build, `VerletPhysicsSystem::DoParallelUpdate`
at `0x00d6c0a0` reads the current `WorldStateComponent`'s `wind` float
before dispatching physics work. It loads the cached component pointer
from `0x01328004` and reads offset `0x78` without a null check at
`0x00d6c15f`. A live test with a missing world state produced an access
violation reading address `0x78`. The dump identifies the failing
system and immediate cause. Identifying who cleared the world state
requires evidence from before the physics update.

`EntityKill` marks an entity and its children for later removal. If a
script starts that process, the native stack at component teardown can
have lost the script's call frame. Correlate the removal with evidence
recorded when the request was made.

## Capturing mod context

For mod attribution, record the active Lua source and the mod that owns the
callback before it enters engine code. Keep those as separate fields since a
script can load code from another mod or run appended Lua. Associate the
records with the thread ID. The same approach works for engine system names
at update entry points. A dump shows where the process failed, while those
records show what was running immediately beforehand. Earlier memory
corruption can make the fault site far removed from the code that caused it.

Noita's bundled LuaJIT supports its C-call wrapper mode. The wrapper gets
the live Lua state and the native function pointer before entering an
external C function. An observer can use that boundary to cover engine
bindings as a group and inspect the Lua caller while its stack is intact.
Each independent Lua VM needs its own installation. Both checked Windows
builds import `lua_pcall` and `lua_close` from `lua51.dll`, so a
recorder can install at the first protected call and discard its VM
tracking on close. The target address can be checked against the
loaded executable's code sections to identify game calls. Direct FFI
calls bypass LuaJIT's C-function wrapper.
[LuaJIT's C API reference](https://luajit.org/ext_c_api.html#mode_wrapcfunc)

Input polling can dominate a recent-call trace. The `InputIsKey*`
bindings check the key index and read input state. Other queries
can have side effects or reach fragile paths. For example,
`EntityGetComponent*` can publish component IDs on demand and
`ComponentGetValue2` can dispatch to a component-specific virtual
method. `InputGetMousePosOnScreen` reaches a renderer pointer.
Review each binding before filtering its calls.

Profiling labels and MSVC RTTI in the executable can supply partial
native names without debug symbols. A recorder can index these before
the fault, then resolve the instruction pointer and return addresses
from a bounded copy of the faulting thread's stack. Frame-pointer
chains can skip optimized functions.

Engine systems give the most useful frame names. Every system class is
created through `SystemManager::TSystemAutoCreator<System>`, so RTTI
lists them by name, and slot 3 (`+0xC`) of a system's vftable is its
update ([finding things without fixed addresses](patching.md#finding-things-without-fixed-addresses)).
A crash inside a system typically reads: the system's update, then the
engine's system loop, which walks a vector of systems and calls that
slot on each (developer build `0x00591a60`), then `DeathMatch`'s update.
The updated-entity global from the same section can be read at fault
time to name the entity being updated, keeping in mind that not every
update path writes it.

In the shipped `lua51.dll`, LuaJIT's interpreter occupies the code
before the first export (`luaL_callmeta` at `+0x61f0`). A return
address there is the VM calling a C function, which is what sits
between a script and the engine binding it called. The exports also
name the standard library's C functions (`lj_cf_table_sort` and so on).
Between exports there are long unexported stretches, the largest
`0x197b0` bytes after `lua_yield`, so a nearest-export name far from
its export says little.

When guessing function starts from padding, a single `0xCC` before an
aligned address is not enough. In the developer build, `0x006e5a4f` is
the SIB byte of `mov [esp+ecx*8], fs` inside `DeathMatch`'s update,
not padding before a function.

An unrestricted LuaJIT mod can install a small native x86 hook using the
system's existing APIs. Keep Lua execution and symbol lookup outside the
crash callback. It runs on the faulting thread, which may be a worker or
may already be inside Lua. Write a bounded record to a file opened in
advance and preserve the game's own dump path. A top-level exception
filter sees only exceptions that reach it. A vectored handler can record
the first-chance fault on either executable. A handled exception also
passes through that handler, so correlate the record with the process
outcome. Fail-fast termination and hangs need separate diagnostics.
[Microsoft's top-level filter reference](https://learn.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-setunhandledexceptionfilter),
[Microsoft's vectored exception reference](https://learn.microsoft.com/en-us/windows/win32/debug/vectored-exception-handling),
[Microsoft's fail-fast reference](https://learn.microsoft.com/en-us/windows/win32/api/errhandlingapi/)
